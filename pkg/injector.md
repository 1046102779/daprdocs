本文主要讲述有关dapr runtime sidecar如何注入到业务pod中去的。并简单分析相关源码的注入过程。

本文由两部分构成
1. k8s准入控制器介绍，以及提及MutatingAdmissionWebhooks控制器；
2. 简单介绍dapr-system命名空间下**dapr-sidecar-injector**对业务pod的dapr runtime sidecar的注入源码分析

## k8s准入控制器

参考：[新手指南之 Kubernetes 准入控制器](https://zhuanlan.zhihu.com/p/106206871)一文

简单说, 如下图所示：

![API请求处理链路](https://wework.qpic.cn/wwpic/213401_NRyg5RxmRGW3nmB_1612145543/0)

k8s准入控制器可以拦截API请求，然后更改API请求对象，验证API对象或者拒绝该请求。

上面这幅图当yaml资源提交到k8s api server后，会经过Authentication/Authrization(认证/授权), Mutation准入控制器(MutatingAdmissionWebhooks： Webhook01, Webhook02 ...), ObjectSchemeValidation(scheme各个对象验证。如上一节CRDs各个对象的资源注册), Validation准入控制器(验证API请求参数是否符合要求)，最后持久化到ETCD集群中

其中Mutation准入控制器，可以修改API请求参数。 那么在dapr中则使用该MutatingAdmissionWebhooks准入控制器，来实现对Deployment资源写入到ETCD集群中时，去动态添加dapr runtime sidecar启动所需要的image和其他相关参数。来实现业务pod启动时，又多一个dapr runtime sidecar容器。

在dapr-system命名空间下，会有一个dapr-sidecar-injector POD，它会接收所有资源创建时，执行到MutatingAdmissionWebhooks后，发送webhook请求，也就是下面准入控制器中的service: **dapr-sidecar-injector**和路由：**/mutate**请求, 去访问dapr-system命名空间下的dapr-sidecar-injector POD来实现注入过程。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: dapr-sidecar-injector
  labels:
    app: dapr-sidecar-injector
webhooks:
- name: sidecar-injector.dapr.io
  clientConfig:
    service:
      namespace: {{ .Release.Namespace }}
      name: dapr-sidecar-injector
      path: "/mutate"
    caBundle: {{ if $existingSecret }}{{ index $existingSecret.data "ca.crt" }}{{ else }}{{ b64enc$ca.Cert }}{{ end }}
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    resources:
    - pods
    operations:
    - CREATE
  failurePolicy: {{ .Values.webhookFailurePolicy}}
  sideEffects: None
  admissionReviewVersions: ["v1", "v1beta1"]
```

## dapr injector注入POD服务

dapr injector主要是通过dapr-system命名空间下的dapr-sidecar-injector POD来处理Mutation准入控制器发送过来的service/mutate请求。那么本dapr injector服务就是dapr-sidecar-injector POD服务，它用来接收mutate路由请求，来实现dapr runtime sidecar的注入过程。

```golang
// NewInjector 根据配置返回injector实例对象
func NewInjector(authUID string, config Config, daprClient scheme.Interface, kubeClient *kubernetes.Clientset) Injector {
    // dapr sidecar injector提供service http 4000端口服务
    mux := http.NewServeMux()

    i := &injector{
        // config表示dapr runtime sidecar启动需要的配置
        //
        // 如：image，镜像拉取策略、dapr runtime sidecar启动的http/https证书路径, 命名空间
        config: config,
        // 接收http 4000端口请求数据，并进行反序列化
        deserializer: serializer.NewCodecFactory(
            runtime.NewScheme(),
        ).UniversalDeserializer(),
        server: &http.Server{
            Addr:    fmt.Sprintf(":%d", port),
            Handler: mux,
        },
        // 通过.kube/config获取kubeclient对象
        kubeClient: kubeClient,
        // 获取dapr client对象
        daprClient: daprClient,
        // 是指service account所对应的objectmeta.uid
        // 也就是所关注的dapr injector的uid过滤唯一指标
        authUID:    authUID,
    }

    // 处理k8s准入控制器Webhook：/mutate过来的请求
    mux.HandleFunc("/mutate", i.handleRequest)
    return i
}

// Run 启动dapr-sidecar-injector POD，提供/mutate API http 4000端口服务
func (i *injector) Run(ctx context.Context) {
    doneCh := make(chan struct{})

    go func() {
        select {
        case <-ctx.Done():
            log.Info("Sidecar injector is shutting down")
            shutdownCtx, cancel := context.WithTimeout(
                context.Background(),
                time.Second*5,
            )
            defer cancel()
            i.server.Shutdown(shutdownCtx) // nolint: errcheck
        case <-doneCh:
        }
    }()

    log.Infof("Sidecar injector is listening on %s, patching Dapr-enabled pods", i.serv  er.Addr)
    err := i.server.ListenAndServeTLS(i.config.TLSCertFile, i.config.TLSKeyFile)
    if err != http.ErrServerClosed {
        log.Errorf("Sidecar injector error: %s", err)
    }
    close(doneCh)
}

// handleRequest 处理/mutate请求
func (i *injector) handleRequest(w http.ResponseWriter, r *http.Request) {
  var body []byte
  if r.Body != nil {
      if data, err := ioutil.ReadAll(r.Body); err == nil {
          body = data
      }
  }
  // 通过deserializer对象，解析http request body数据
  // 转换成AdmissionReview协议目标数据
  /*
    type AdmissionReview struct {
      metav1.TypeMeta `json:",inline"`
      Request *AdmissionRequest `json:"request,omitempty" protobuf:"bytes,1,opt,name=request"`
      Response *AdmissionResponse `json:"response,omitempty" protobuf:"bytes,2,opt,name=response"`
    }
  */
  var admissionResponse *v1.AdmissionResponse
  var patchOps []PatchOperation
  var err error

  ar := v1.AdmissionReview{}
  _, gvk, err := i.deserializer.Decode(body, nil, &ar)
  // 校验发送过来的请求数据，校验service account的uid与dapr injector的service account的uid是否相同
  // 这个是唯一用来dapr injector准入控制器的过滤指标
  if ar.Request.UserInfo.UID != i.authUID {
      err = errors.Wrapf(err, "unauthorized request")
      log.Error(err)
  // 必须是pod类型时才会做mutation修改
  } else if ar.Request.Kind.Kind != "Pod" {
      err = errors.Wrapf(err, "invalid kind for review: %s", ar.Kind)
      log.Error(err)
  } else {
      // 获取需要修改的资源对象，也就是打补丁，把dapr runtime sidecar运行所需要的资源全部写入到pod yaml文件中。构建成完整的pod yaml。
      patchOps, err = i.getPodPatchOperations(&ar, i.config.Namespace, i.config.S  idecarImage, i.config.SidecarImagePullPolicy, i.kubeClient, i.daprClient)
  }
  // 如果不需要进行pod请求资源修改，则直接返回响应数据
  if len(patchOps) == 0 {
        admissionResponse = &v1.AdmissionResponse{
            Allowed: true,
        }
  } else {
      // 否则，序列化patchOps为json流
      var patchBytes []byte
      patchBytes, err = json.Marshal(patchOps)
      if err != nil {
          admissionResponse = toAdmissionResponse(err)
      } else {
          // 序列化json流，并标识序列化的方法PathTypeJSONPatch JSONPatch
          // 并构建admissionResponse响应数据
          admissionResponse = &v1.AdmissionResponse{
              Allowed: true,
              Patch:   patchBytes,
              PatchType: func() *v1.PatchType {
                  pt := v1.PatchTypeJSONPatch
                  return &pt
              }(),
          }
      }
  }

  // 构建响应数据协议：v1.AdmissionReview{},并填充数据
  // 设置AdmissionReview的Response，以及metav1.TypeMeta, 和service account的UID
  admissionReview := v1.AdmissionReview{}
  if admissionResponse != nil {
      admissionReview.Response = admissionResponse
      if ar.Request != nil {
          admissionReview.Response.UID = ar.Request.UID
          admissionReview.SetGroupVersionKind(*gvk)
      }
  }

  log.Infof("ready to write response ...")
  // 最后通过json序列化为respBytes，并通过w.Write写入到http响应包并返回给上游
  respBytes, err := json.Marshal(admissionReview)
  ......
  w.Header().Set("Content-Type", "application/json")
  if _, err := w.Write(respBytes); err != nil {
      log.Error(err)
  } else {
      monitoring.RecordSuccessfulSidecarInjectionCount(diagAppID)
  }
}

// getPodPatchOperations 获取PathchOperation列表. 这个也就是dapr runtime sidecar所需要的操作Path列表
func (i *injector) getPodPatchOperations(ar *v1.AdmissionReview,
  namespace, image, imagePullPolicy string, kubeClient *kubernetes.Clientset, daprClient scheme.Interface) ([]PatchOperation, error) {
  // 把AdmissionReview中的Object.Raw值，通过json反序列化为v1.Pod
  req := ar.Request
  var pod corev1.Pod
  if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
      errors.Wrap(err, "could not unmarshal raw object")
      return nil, err
  }

  // 校验pod资源中的annotations的dapr.io/enabled变量值是否开启
  // 或者
  // 校验pod资源中的spec.containers, 判断name是否存在容器名daprd的变量值
  //
  // 通过上面两个annotations和spec.containers列表中的name，来判断是否需要开启dapr runtime sidecar
  if !isResourceDaprEnabled(pod.Annotations) || podContainsSidecarContainer(&pod) {
      return nil, nil
  }

  // 通过pod中的annotations，获取参数名为dapr.io/app-id的参数值
  // 如果不存在，则直接获取pod的名称，作为dapr runtime sidecar的appid的参数
  // 获取业务pod启动时指定的--app-id
  id := getAppID(pod)
  // 验证id是否符合k8s的命名规范
  err := validation.ValidateKubernetesAppID(id)
  if err != nil {
      return nil, err
  }
  // 标识访问dapr-system命名空间下的dapr-placement pod服务，它直接通过k8s service dns集群内解析获取POD IP地址
  // 比如：dapr-system命名空间下的placement POD服务，最终的DNS为
  // DNS: dapr-placement-server.dapr-system.svc.cluster.local 端口：50005
  // DNS: dapr-sentry.dapr-system.svc.cluster.local 端口：80
  // DNS: dapr-api.dapr-system.cluster.local 端口：80
  // placement server提供actor服务
  placementAddress := fmt.Sprintf("%s:50005", getKubernetesDNS(placementService, namespace))
  // sentry server证书提供者
  sentryAddress := fmt.Sprintf("%s:80", getKubernetesDNS(sentryService, namespace))
  // dapr api提供外部服务
  apiSrvAddress := fmt.Sprintf("%s:80", getKubernetesDNS(apiAddress, namespace))

   var trustAnchors string
   var certChain string
   var certKey string
   var identity string

   mtlsEnabled := mTLSEnabled(daprClient)
   if mtlsEnabled {
       trustAnchors, certChain, certKey = getTrustAnchorsAndCertChain(kubeClient, namespace)
       identity = fmt.Sprintf("%s:%s", req.Namespace, pod.Spec.ServiceAccountName)
   }

   // service account挂载地址：/var/run/secrets/kubernetes.io/serviceaccount
   tokenMount := getTokenVolumeMount(pod)
   // 构建daprd container资源配置对象
   sidecarContainer, err := getSidecarContainer(pod.Annotations, id, image, imagePullPolicy, req.Namespace, apiSrvAddress, placementAddress, tokenMount, trustAnchors, certCh  ain, certKey, sentryAddress, mtlsEnabled, identity)
   if err != nil {
       return nil, err
   }
   // 校验传入的http请求中的POD的spec.containers容器列表参数中是否含有其他容器镜像
   // 如果不存在，则表示该业务pod的sidecar只有daprd和业务本身
   // 否则，表示还有其他镜像，比如：日志镜像、CL5镜像等
   //
   // path表示pod yaml文件中资源位置，比如/spec/containers/-
   // value表示资源位置的资源值，比如：/spec/containers/env的值
   /*
      DAPR_HTTP_PORT: 3500
      DAPR_GRPC_PORT: 50001
   */
   patchOps := []PatchOperation{}
    envPatchOps := []PatchOperation{}
    var path string
    var value interface{}
    if len(pod.Spec.Containers) == 0 {
        path = containersPath
        value = []corev1.Container{*sidecarContainer}
    } else {
        // 如果除了daprd runtime sidecar镜像外，还存在其他镜像，则需要把所有的镜像都注入两个环境变量
        //  这两个环境变量分别是: dapr runtime http 3500和dapr runtime grpc 50001
        //  这业务POD中的每个container都应该通过dapr runtime sidecar访问外部
        envPatchOps = addDaprEnvVarsToContainers(pod.Spec.Containers)
        path = "/spec/containers/-"
        value = sidecarContainer
    }

    // 首先把daprd runtime sidecar容器注入业务POD中
    patchOps = append(
        patchOps,
        PatchOperation{
            Op:    "add",
            Path:  path,
            Value: value,
        },
    )
    // 然后把该业务POD中的其他镜像，包括业务本身container等镜像，添加与daprd runtime sidecar相关的服务环境变量
    // 分别是http 3500和grpc 50001
    patchOps = append(patchOps, envPatchOps...)

    return patchOps, nil
  }

// addDaprEnvVarsToContainers 把业务pod中的spec.containers遍历，分别添加上访问dapr runtime sidecar的环境变量
// DAPR_HTTP_PORT http 3500和DAPR_GRPC_PORT grpc 50001
// 注意：如果.specs.containers已经存在container引用该环境变量，则不修改和覆盖
func addDaprEnvVarsToContainers(containers []corev1.Container) []PatchOperation {
    portEnv := []corev1.EnvVar{
        {
            Name:  userContainerDaprHTTPPortName,
            Value: strconv.Itoa(sidecarHTTPPort),
        },
        {
            Name:  userContainerDaprGRPCPortName,
            Value: strconv.Itoa(sidecarAPIGRPCPort),
        },
    }
    envPatchOps := []PatchOperation{}
    for i, container := range containers {
        path := fmt.Sprintf("%s/%d/env", containersPath, i)
        patchOps := getEnvPatchOperations(container.Env, portEnv, path)
        envPatchOps = append(envPatchOps, patchOps...)
    }
    return envPatchOps
}
```

最终业务POD的.spec.containers会构成建成：
1. 增加容器daprd runtime sidecar到业务POD中
2. 为该业务POD中的所有其他container增加，访问该POD中的dapr runtime sidecar容器所需要的两个环境变量，分别是DAPR_HTTP_PORT:3500和DAPR_GRPC_PORT:50001

第二点通过k8s准入控制器修改数据后的响应数据有关podpatches .spec.containers的修改，如下数据协议格式所示：

```json
[{
  "op": "add",
  "path": "/spec/containers/-",
  "value": "json(sidecarContainer)"
},
{
  "op": "add",
  "path":"/spec/containers/env",
  "value": [
    {
      "name": "DAPR_HTTP_PORT",
      "value": "3500"
    },
    {
      "name": "DAPR_GRPC_PORT",
      "value": "50001"
    }
  ]
}]
```

### 注入dapr runtime sidecar容器

dapr runtime sidecar注入到业务POD中，主要包括业务POD自身容器的环境变量的注入过程(dapr http和grpc，保证所有业务容器都可以通过dapr runtime sidecar访问外部服务)，以及dapr runtime sidecar的注入过程；其中后者的注入包括了对业务服务治理方面的参数设置。

```golang
//  getSidecarContainer 通过参数构建dapr runtime sidecar容器参数对象
func getSidecarContainer(annotations map[string]string, id, daprSidecarImage, imagePullPolicy, namespace, controlPlaneAddress, placementServiceAddress string, tokenVolumeMount *corev1.VolumeMount, trustAnchors, certChain, certKey, sentryAddress string, mtlsEnabled bool, identity string) (*corev1.Container, error) {
  // 通过pod中的annotations获取一些参数值：
  // 1. dapr.io/app-port业务服务端口
  // 2. dapr.io/metrics-port
  // 3. dapr.io/app-max-concurrency业务POD的最大处理并发量
  // 4. dapr.io/app-ssl SSL校验
  // 5. dapr.io/http-max-request-size 每个请求最大处理的数据量size
  appPort, err := getAppPort(annotations)
  appPortStr := ""
  if appPort > 0 {
      appPortStr = fmt.Sprintf("%v", appPort)
  }
  metricsPort := getMetricsPort(annotations)
  maxConcurrency, err := getMaxConcurrency(annotations)
  sslEnabled := appSSLEnabled(annotations)
  requestBodySize, err := getMaxRequestBodySize(annotations)

  // 获取dapr runtime sidecar镜像拉取策略={Always, Never和IfNotPresent}
  pullPolicy := getPullPolicy(imagePullPolicy)
  // 获取dapr runtime sidecar容器的健康探针: http 3500端口服务，router: /v1.0/healthz
  httpHandler := getProbeHTTPHandler(sidecarHTTPPort, apiVersionV1, sidecarHealthzPath)
  allowPrivilegeEscalation := false

  // 构建dapr runtime sidecar所需要的容器资源配置yaml
  c := &corev1.Container{
      Name:            sidecarContainerName, // 容器名：daprd
      Image:           daprSidecarImage, // 镜像地址
      ImagePullPolicy: pullPolicy, // 镜像拉取策略
      SecurityContext: &corev1.SecurityContext{
          AllowPrivilegeEscalation: &allowPrivilegeEscalation,
      },
      // 业务POD对外暴露的端口服务，也即是dapr runtime sidecar暴露的dapr service
      // 包括：http 3500、grpc 50001， 内部grpc 50002和metrics 9090
      Ports: []corev1.ContainerPort{
          {
              ContainerPort: int32(sidecarHTTPPort),
              Name:          sidecarHTTPPortName,
          },
          {
              ContainerPort: int32(sidecarAPIGRPCPort),
              Name:          sidecarGRPCPortName,
          },
          {
              ContainerPort: int32(sidecarInternalGRPCPort),
              Name:          sidecarInternalGRPCPortName,
          },
          {
              ContainerPort: int32(metricsPort),
              Name:          sidecarMetricsPortName,
          },
      },
      // daprd runtime sidecar容器服务启动的命令：/darpd
      Command: []string{"/daprd"},
      // daprd 运行需要的环境变量
      Env: []corev1.EnvVar{
          {
              Name: utils.HostIPEnvVar,
              ValueFrom: &corev1.EnvVarSource{
                  FieldRef: &corev1.ObjectFieldSelector{
                      FieldPath: "status.podIP",
                  },
              },
          },
          ...
      },
      // daprd runtime sidecar容器运行所需要的参数
      Args: []string{
              "--mode", "kubernetes", // 运行模式k8s
              "--dapr-http-port", fmt.Sprintf("%v", sidecarHTTPPort), // http 3500
              "--dapr-grpc-port", fmt.Sprintf("%v", sidecarAPIGRPCPort), // grpc 50001
              "--dapr-internal-grpc-port", fmt.Sprintf("%v", sidecarInternalGRPCPort), // grpc 50002
              "--app-port", appPortStr, // 业务提供的服务端口
              "--app-id", id, // 业务ID
              "--control-plane-address", controlPlaneAddress, // dapr runtime sidecar API Server
              "--app-protocol", getProtocol(annotations), // dapr.io/app-protocol应用服务端口协议
              "--placement-host-address", placementServiceAddress, // dapr placement服务地址列表
              "--config", getConfig(annotations), // dapr.io/config配置
              "--log-level", getLogLevel(annotations), // dapr.io/log-level日志级别
              "--app-max-concurrency", fmt.Sprintf("%v", maxConcurrency), // 业务POD最大并发量
              "--sentry-address", sentryAddress, // 证书提供者服务
              "--metrics-port", fmt.Sprintf("%v", metricsPort), // metrics服务
              "--dapr-http-max-request-size", fmt.Sprintf("%v", requestBodySize), // 业务POD请求最大处理数据量
          },
          // 就绪探针，当业务POD启动时，判断该服务是否可以正常提供服务的校验
          //
          // 这里的就绪探针和存活探针都是使用的3500 /v1.0/healthz路由服务作为入口
          ReadinessProbe: &corev1.Probe{
              Handler:             httpHandler,
              // annotations中的dapr.io/sidecar-readiness-probe-delay-seconds参数
              // 初始化默认值：3s
              InitialDelaySeconds: getInt32AnnotationOrDefault(annotations, daprReadinessProbeDelayKey, defaultHealthzProbeDelaySeconds),
              // annotations中的dapr.io/sidecar-readiness-probe-timeout-seconds参数
              // 超时默认值：3s
              TimeoutSeconds:      getInt32AnnotationOrDefault(annotations, daprReadinessProbeTimeoutKey, defaultHealthzProbeTimeoutSeconds),
              // annotations中的dapr.io/sidecar-readiness-probe-period-seconds参数
              // 探针周期默认值：6s
              PeriodSeconds:       getInt32AnnotationOrDefault(annotations, daprReadinessProbePeriodKey, defaultHealthzProbePeriodSeconds),
              // annotations中的dapr.io/sidecar-readiness-probe-threshold参数
              // 默认值： 3，表示如果3次探测都失败，表示该业务POD不可用
              FailureThreshold:    getInt32AnnotationOrDefault(annotations, daprReadinessProbeThresholdKey, defaultHealthzProbeThreshold),
          },
          // 存活探针，也就是常规的健康检查
          LivenessProbe: &corev1.Probe{
              Handler:             httpHandler,
              // dapr.io/sidecar-liveness-probe-delay-seconds
              // 初始化默认值：3s
              InitialDelaySeconds: getInt32AnnotationOrDefault(annotations, daprLivenessProbeDelayKey, defaultHealthzProbeDelaySeconds),
              // dapr.io/sidecar-liveness-probe-timeout-seconds
              // 超时默认值：3s
              TimeoutSeconds:      getInt32AnnotationOrDefault(annotations, daprLivenessProbeTimeoutKey, defaultHealthzProbeTimeoutSeconds),
              // dapr.io/sidecar-liveness-probe-period-seconds
              // 探测周期默认值：6s
              PeriodSeconds:       getInt32AnnotationOrDefault(annotations, daprLivenessProbePeriodKey, defaultHealthzProbePeriodSeconds),
              // dapr.io/sidecar-liveness-probe-threshold
              // 默认值：3，最大健康检查次数3，如果3次连续失败，则表示服务不可用
              FailureThreshold:    getInt32AnnotationOrDefault(annotations, daprLivenessProbeThresholdKey, defaultHealthzProbeThreshold),
          },
      }

      // 如果service account有指定，则传入到container中
      if tokenVolumeMount != nil {
          c.VolumeMounts = []corev1.VolumeMount{
              *tokenVolumeMount,
          }
      }

      // 业务日志上报json是否开启：dapr.io/log-as-json
      if logAsJSONEnabled(annotations) {
          c.Args = append(c.Args, "--log-as-json")
      }

      // 是否开启性能采样
      if profilingEnabled(annotations) {
          c.Args = append(c.Args, "--enable-profiling")
      }

      // 证书mtls获取
      if mtlsEnabled && trustAnchors != "" {
          c.Args = append(c.Args, "--enable-mtls")
          c.Env = append(c.Env, corev1.EnvVar{
              Name:  certs.TrustAnchorsEnvVar,
              Value: trustAnchors,
          },
              corev1.EnvVar{
                  Name:  certs.CertChainEnvVar,
                  Value: certChain,
              },
              corev1.EnvVar{
                  Name:  certs.CertKeyEnvVar,
                  Value: certKey,
              },
              corev1.EnvVar{
                  Name:  "SENTRY_LOCAL_IDENTITY",
                  Value: identity,
              })
      }

      // ssl证书
      if sslEnabled {
          c.Args = append(c.Args, "--app-ssl")
      }

      // dapr.io/api-token-secret获取secret
      secret := getAPITokenSecret(annotations)
      if secret != "" {
          c.Env = append(c.Env, corev1.EnvVar{
              Name: auth.APITokenEnvVar,
              ValueFrom: &corev1.EnvVarSource{
                  SecretKeyRef: &corev1.SecretKeySelector{
                      Key: "token",
                      LocalObjectReference: corev1.LocalObjectReference{
                          Name: secret,
                      },
                  },
              },
          })
      }

      // dapr.io/app-token-secret获取app secret
      appSecret := GetAppTokenSecret(annotations)
      if appSecret != "" {
          c.Env = append(c.Env, corev1.EnvVar{
              Name: auth.AppAPITokenEnvVar,
              ValueFrom: &corev1.EnvVarSource{
                  SecretKeyRef: &corev1.SecretKeySelector{
                      Key: "token",
                      LocalObjectReference: corev1.LocalObjectReference{
                          Name: appSecret,
                      },
                  },
              },
          })
      }
      // 获取业务POD对资源的要求, 资源类型：cpu和内存 包括：limit和request
      // dapr.io/sidecar-cpu-limit
      // dapr.io/sidecar-memory-limit
      // dapr.io/sidecar-cpu-request
      // dapr.io/sidecar-memory-request
      resources, err := getResourceRequirements(annotations)
     if err != nil {
         log.Warnf("couldn't set container resource requirements: %s. using defaults", e  rr)
     }
     // 并写入到container的resources
     if resources != nil {
         c.Resources = *resources
     }
     return c, nil
}
```

通过前面dapr injector整个注入源码分析，知道了MutatingAdmissionWebhooks webhook请求返回的PatchPod列表，来增加pod的其他资源注入或者修改，使得业务POD部署运行符合资源预期。

## 总结

通过对dapr injector源码分析，我们可以了解到dapr-system命名空间下的dapr-injector POD是通过注入k8s准入控制器，在提交资源对象到etcd集群之前，进行MutatingAdmissionWebhooks webhook请求，访问dapr-injector服务的http 4000端口, 路由为/mutate, 修改请求的资源对象。该过程包括dapr runtime sidecar容器的注入，以及其他容器参数访问dapr runtime sidecar的grpc和http端口服务。目的就是，所有业务POD中的所有容器都通过该dapr runtime sidecar容器暴露给外部访问。

而dapr runtime sidecar的注入到业务POD中的整个过程，包含了非常多的有关业务POD的参数控制， 包括：
1. 证书提供；
2. 业务服务流控；
3. 资源配额设置；
4. 业务服务性能开启；
5. 业务日志级别设置和json化；
6. 就绪和探活包检测及各个参数设置；
7. placement server和sentry server的访问服务端口配置;
8. 业务POD一次接收请求的最大包；
9. 指标上报的参数配置metrics；
10. 设置业务服务提供的端口和端口协议；

以上基本上把业务服务所关注的整个服务治理相关参数全部配置好了。减少业务开发人员针对这一块的开发工作。
