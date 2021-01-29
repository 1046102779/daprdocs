k8s operator是CoreOS团队(现已被RetHat收购)提出的，主要用于对k8s CRDs资源变化的监听，并根据配置资源的预期，来决策、调度和部署资源。k8s支持的资源比较有限，比如Deployment、Service、Ingress等等，但是它并不能支持各类资源的组合一件部署、以及k8s除了内置资源的其他资源定义，并不能识别. 当k8s api server监听到etcd资源发生变更后，就会给到各个k8s controllers进行相关的资源处理。

dapr operator的主要用途是，针对k8s CRDs(custom resource definitions 自定义资源), 创建DaprController控制器，去监听资源变化, 进行资源部署，包括是否需要进行dapr service部署。所有的CRDs基本都是通过资源的spec的anntotions注释部分，携带特征参数资源。如：**dapr.io/enabled**, **dapr.io/app-id**, **dapr.io/metrics-port**等.

其中所有的CRDs的CustomController，k8s都只对其暴露了一个回调实现API——**Reconcile**. 这个就是各个k8s controllers，包括内置和自定义实现的controller。比如：内置DeploymentController和自定义实现的DaprController等。并针对自己感兴趣的资源进行调度和相关资源创建、变更等等。以达到服务运行的预期

dapr operator其实是一个dapr系统服务，它存在于dapr-system命名空间下的dapr-operator pod服务中，并注册和订阅自己所关注的资源对象，以及为业务pod创建dapr service服务，并暴露一系列端口服务给外部使用。同时还对所有业务pod提供获取CRDs资源对象服务。这个是通过grpc 3500端口服务提供。


```golang
// https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile#Reconciler
//
// Reconciler通过创建，更新或删除Kubernetes对象或通过更改集群外部系统（例如cloudproviders，github等）来为特定资源实现Kubernetes API。
//
// 当k8s api server通过watch etcd发生资源变化后，则会通过各个controller订阅的资源，调用各个Controller的Reconcile API，进行资源的创建和变更。
type Reconciler interface {
	Reconcile(context.Context, Request) (Result, error)
}
```

## dapr operator

dapr operator主要有两个作用：
- Reconcile API用于dapr controller接收API Server发送过来的资源变更请求, 进行资源调度和部署；
- 通过k8s api server的client，获取dapr configuration配置、Components组件列表(components-contrib)和订阅者列表。并通过grpc提供：**GetConfiguration**, **ListComponents**, **ListSubscriptions**和**ComponentUpdate**四个API服务。

针对第二点，主要是把提交的配置资源数据同步给dapr runtime。

```golang
// DaprHandler 用于处理Dapr CRDs的生命周期，也就是k8s operator CRDs
type DaprHandler struct {
    // Manager 用于初始化依赖，如：缓存、k8s api server client和其他依赖, 并通过该Manager创建DaprController
    mgr ctrl.Manager

    // k8s api server client通过k8s api server获取etcd存储的数据。
    client.Client
    // Scheme 是对所有API资源对象进行序列化和反序列化
    Scheme *runtime.Scheme
}

// NewDaprHandler 创建DaprHandler对象
func NewDaprHandler(mgr ctrl.Manager) *DaprHandler {
    return &DaprHandler{
        // Manager用于初始化依赖和启动DaprController
        mgr: mgr,

        // 通过Manager获取k8s api server client
        Client: mgr.GetClient(),
        // 通过Manager获取Scheme，来对资源对象进行序列化和反序列化
        Scheme: mgr.GetScheme(),
    }
}

// DaprHandler Init初始化Manager
func (h *DaprHandler) Init() error {
    // 注册设置Service的字段索引匹配规则，来找到Service的Deployment实例名称Name
    if err := h.mgr.GetFieldIndexer().IndexField(
        context.TODO(),
        &corev1.Service{}, daprServiceOwnerField, func(rawObj client.Object) []string {
            svc := rawObj.(*corev1.Service)
            owner := meta_v1.GetControllerOf(svc)
            if owner == nil || owner.APIVersion != appsv1.SchemeGroupVersion.String() |  | owner.Kind != "Deployment" {
                return nil
            }
            return []string{owner.Name}
        }); err != nil {
        return err
    }

    // 构建Manager所需要的Builder，以及定义DaprController关注的对象Deployment和定义Service
    // 并通过Builder构建DaprController，订阅所关注的对象
    return ctrl.NewControllerManagedBy(h.mgr).
        For(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Complete(h)
}

/*
apiVersion: v1
kind: Service
metadata:
  ...
  ownerReferences:
  - apiVersion: apps/v1
    controller: true
    blockOwnerDeletion: true
    kind: Deployment
    name: dapr-appid
    uid: d9607e19-f88f-11e6-a518-42010a800195
  ...
*/

// daprServiceName 用于返回注入dapr POD的实例名称appid-dapr
func (h *DaprHandler) daprServiceName(appID string) string {
    return fmt.Sprintf("%s-dapr", appID)
}

// Reconcile 用于接收订阅的资源配置，并做相关资源部署和调度动作
func (h *DaprHandler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, er  ror) {
    var deployment appsv1.Deployment
    expectedService := false
    // 通过Namespace/Name，获取Deployment资源, 如果不存在，则判断是否deployment删除了?
    if err := h.Get(ctx, req.NamespacedName, &deployment); err != nil {
        if apierrors.IsNotFound(err) {
            log.Debugf("deployment has be deleted, %s", req.NamespacedName)
        } else {
            log.Errorf("unable to get deployment, %s, err: %s", req.NamespacedName, err  )
            return ctrl.Result{}, err
        }
    } else {
        // 如果存在，但是通过DeletionTimestamp参数值，判断deployment是否正在删除中
        if deployment.DeletionTimestamp != nil {
            log.Debugf("deployment is being deleted, %s", req.NamespacedName)
            return ctrl.Result{}, nil
        }
        // 如果Deployment已存在，通过Deployment资源配置对象中，获取annotations注释配置中dapr.io/enabled,
        // 判断该值是否为true。如果为true，表示需要该业务pod需要启用dapr runtime container。
        expectedService = h.isAnnotatedForDapr(&deployment)
    }

    // 如果dapr.io/enabled=true, 则表示需要创建service，也就是deployment的service。这个service是指访问dapr container的服务。
    if expectedService {
        if err := h.ensureDaprServicePresent(ctx, req.Namespace, &deployment); err != n  il {
            return ctrl.Result{Requeue: true}, err
        }
    } else {
        // 如果deployment的dapr service已存在，则需要删除service
        if err := h.ensureDaprServiceAbsent(ctx, req.NamespacedName); err != nil {
            return ctrl.Result{Requeue: true}, err
        }
    }
    return ctrl.Result{}, nil
}

// ensureDaprServicePresent 用于创建dapr service
//
// 因为业务pod只暴露dapr service，不会暴露业务pod的container。
func (h *DaprHandler) ensureDaprServicePresent(ctx context.Context, namespace string, deployment *appsv1.Deployment) error {
    // 通过deployment的annotations中的dapr.io/app-id
    // 并校验appid是否合法.
    // 因为这个appid将会用于dapr runtime的dapr service name
    appID := h.getAppID(deployment)
    // 校验appid是否合法
    err := validation.ValidateKubernetesAppID(appID)
    if err != nil {
        return err
    }

    // 通过service的namespace和name(appid-dapr)，获取dapr service
    mayDaprService := types.NamespacedName{
        Namespace: namespace,
        Name:      h.daprServiceName(appID),
    }
    var daprSvc corev1.Service
    // 这个就是通过k8s api service client获取dapr service
    if err := h.Get(ctx, mayDaprService, &daprSvc); err != nil {
        // 如果dapr service不存在，则需要为deployment创建dapr service
        if apierrors.IsNotFound(err) {
            log.Debugf("no service for deployment found, deployment: %s/%s", namespace,   deployment.Name)
            // 创建dapr service
            return h.createDaprService(ctx, mayDaprService, deployment)
        }
        log.Errorf("unable to get service, %s, err: %s", mayDaprService, err)
        return err
    }
    return nil
}

// createDaprService 为业务deployment创建dapr service.
func (h *DaprHandler) createDaprService(ctx context.Context, expectedService types.NamespacedName, deployment *appsv1.Deployment) error {
    // 通过deployment的annotations dapr.io/app-id获取appid
    appID := h.getAppID(deployment)
    // 通过deployment的annotations dapr.io/metrics-port获取指标上报服务端口
    metricsPort := h.getMetricsPort(deployment)

    // 构建一个dapr service所需要的结构对象。
    // {metadata, spec, status}
    service := &corev1.Service{
        // metadata: {name, namespace, label, annotations}
        ObjectMeta: meta_v1.ObjectMeta{
            Name:      expectedService.Name,
            Namespace: expectedService.Namespace,
            Labels:    map[string]string{daprEnabledAnnotationKey: "true"},
            Annotations: map[string]string{
                "prometheus.io/scrape": "true",
                "prometheus.io/port":   strconv.Itoa(metricsPort),
                "prometheus.io/path":   "/",
                appIDAnnotationKey:     appID,
            },
        },
        // spec: {selector, clusterIP, ports}
        // 其中：selector: 表示对业务pod中deployment的name选择，也就是对deployment的选择。
        // ports 提供多端口服务，有80端口。
        Spec: corev1.ServiceSpec{
            // service对deployment的选择，在Reconcile API会动态新增OwnerReference的.metadata.controller
            // 也就是service通过deployment的name来选择后端资源
            Selector:  deployment.Spec.Selector.MatchLabels,
            ClusterIP: clusterIPNone,
            Ports: []corev1.ServicePort{
                // dapr-http port: 80 target 3500
                {
                    Protocol:   corev1.ProtocolTCP,
                    Port:       80,
                    TargetPort: intstr.FromInt(daprSidecarHTTPPort),
                    Name:       daprSidecarHTTPPortName,
                },
                // dapr-grpc port: 50001: target: 50001
                {
                    Protocol:   corev1.ProtocolTCP,
                    Port:       int32(daprSidecarAPIGRPCPort),
                    TargetPort: intstr.FromInt(daprSidecarAPIGRPCPort),
                    Name:       daprSidecarAPIGRPCPortName,
                },
                // dapr-internal port: 50002 target: 50002
                {
                    Protocol:   corev1.ProtocolTCP,
                    Port:       int32(daprSidecarInternalGRPCPort),
                    TargetPort: intstr.FromInt(daprSidecarInternalGRPCPort),
                    Name:       daprSidecarInternalGRPCPortName,
                },
                // dapr-metrics port: 9090 target: 9090
                {
                    Protocol:   corev1.ProtocolTCP,
                    Port:       int32(metricsPort),
                    TargetPort: intstr.FromInt(metricsPort),
                    Name:       daprSidecarMetricsPortName,
                },
            },
        },
    }
    // 设置控制关系，实际上是给pod添加了.metadata.ownerReferences字段
    //
    // 设置deployment与pod之间的生命周期关联
    if err := ctrl.SetControllerReference(deployment, service, h.Scheme); err != nil {
        return err
    }
    // 通过k8s api server client创建service服务对象
    if err := h.Create(ctx, service); err != nil {
        log.Errorf("unable to create Dapr service for deployment, service: %s, err: %s"  , expectedService, err)
        return err
    }
    log.Debugf("created service: %s", expectedService)
    monitoring.RecordServiceCreatedCount(appID)
    return nil
}

// ensureDaprServiceAbsent 获取deploymentKey映射的services是否已经存在
// 如果已经存在，则删除services
func (h *DaprHandler) ensureDaprServiceAbsent(ctx context.Context, deploymentKey types.NamespacedName) error {
    var services corev1.ServiceList
    // 通过namespace和name，k8s api server client获取services列表
    if err := h.List(ctx, &services,
        client.InNamespace(deploymentKey.Namespace),
        client.MatchingFields{daprServiceOwnerField: deploymentKey.Name}); err != nil {          log.Errorf("unable to list services, err: %s", err)
        return err
    }
    // 遍历每个service，并删除所有service
    for i := range services.Items {
        svc := services.Items[i]
        log.Debugf("deleting service: %s/%s", svc.Namespace, svc.Name)
        if err := h.Delete(ctx, &svc, client.PropagationPolicy(meta_v1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
            log.Errorf("unable to delete svc: %s/%s, err: %s", svc.Namespace, svc.Name,   err)
        } else {
            log.Debugf("deleted service: %s/%s", svc.Namespace, svc.Name)
            appID := svc.Annotations[appIDAnnotationKey]
            monitoring.RecordServiceDeletedCount(appID)
        }
    }
    return nil
}
```

因为是CRDs自定义资源，需要注册到Scheme中，用于序列化和反序列化API对象. dapr operator需要识别三种CRDs，分别是：component, configuration和subscription

### CRDs：component

```golang
// 注册components到scheme，包括两种：component和component list
func addKnownTypes(scheme *runtime.Scheme) error {
    scheme.AddKnownTypes(
        // groupVersion: dapr.io/v1alpha1
        SchemeGroupVersion,
        &Component{},
        &ComponentList{},
    )
    metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
    return nil
}

// Component的构成={kind, apiversion, metadata, spec, auth, scopes}
// CRDs Component描述来Dapr component类型
type Component struct {
    metav1.TypeMeta `json:",inline"`
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`
    // +optional
    Spec ComponentSpec `json:"spec,omitempty"`
    // +optional
    Auth `json:"auth,omitempty"`
    // +optional
    Scopes []string `json:"scopes,omitempty"`
}

// ComponentSpec={type, version, ignoreErrors, metadata, initTimout}
type ComponentSpec struct {
    Type string `json:"type"`
    // +optional
    Version string `json:"version"`
    // +optional
    IgnoreErrors bool           `json:"ignoreErrors"`
    Metadata     []MetadataItem `json:"metadata"`
    // +optional
    InitTimeout string `json:"initTimeout"`
}

// MetadataItem={name, value, secretKeyRef}
type MetadataItem struct {
    Name string `json:"name"`
    // +optional
    Value DynamicValue `json:"value,omitempty"`
    // +optional
    SecretKeyRef SecretKeyRef `json:"secretKeyRef,omitempty"`
}

// SecretKeyRef={name, key}
type SecretKeyRef struct {
    Name string `json:"name"`
    Key  string `json:"key"`
}
```


```yaml
----------------component.yaml-----------------
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: sample-topic
spec:
  type: bindings.kafka
  metadata:
  # Kafka broker connection setting
  - name: brokers
    value: localhost:9092
  # consumer configuration: topic and consumer group
  - name: topics
    value: sample
  - name: consumerGroup
    value: group1
  # publisher configuration: topic
  - name: publishTopic
    value: sample
  - name: authRequired
    value: "false"
```

### CRDs: configuration

```golang
// Configuration={kind, apiversion, metadata, spec}
//
// Configuration描述来configuration资源协议
type Configuration struct {
    metav1.TypeMeta `json:",inline"`
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`
    // +optional
    Spec ConfigurationSpec `json:"spec,omitempty"`
}

// ConfigurationSpec={httpPipeline, tracing, metric, mtls, secrets, accessControl}
type ConfigurationSpec struct {
    // +optional
    HTTPPipelineSpec PipelineSpec `json:"httpPipeline,omitempty"`
    // +optional
    TracingSpec TracingSpec `json:"tracing,omitempty"`
    // +kubebuilder:default={enabled:true}
    MetricSpec MetricSpec `json:"metric,omitempty"`
    // +optional
    MTLSSpec MTLSSpec `json:"mtls,omitempty"`
    // +optional
    Secrets SecretsSpec `json:"secrets,omitempty"`
    // +optional
    AccessControlSpec AccessControlSpec `json:"accessControl,omitempty"`
}

// SecretsSpec is the spec for secrets configuration
type SecretsSpec struct {
    Scopes []SecretsScope `json:"scopes"`
}

// SecretsScope defines the scope for secrets
type SecretsScope struct {
    // +optional
    DefaultAccess string `json:"defaultAccess,omitempty"`
    StoreName     string `json:"storeName"`
    // +optional
    AllowedSecrets []string `json:"allowedSecrets,omitempty"`
    // +optional
    DeniedSecrets []string `json:"deniedSecrets,omitempty"`
}
```

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
spec:
  tracing:
    samplingRate: "1"
```

### CRDs: Subscription

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: myevent-subscription
spec:
  topic: deathStarStatus
  route: /dsstatus
  pubsubname: pubsub
scopes:
- app1
- app2
```

```golang
// subscription={kind, apiversion, metadata, spec, scopes}
// Subscription描述来订阅pub/sub事件
type Subscription struct {
    metav1.TypeMeta `json:",inline"`
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              SubscriptionSpec `json:"spec,omitempty"`
    // +optional
    Scopes []string `json:"scopes,omitempty"`
}

// SubscriptionSpeci
type SubscriptionSpec struct {
    Topic      string `json:"topic"`
    Route      string `json:"route"`
    Pubsubname string `json:"pubsubname"`
}

type SubscriptionList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata"`

    Items []Subscription `json:"items"`
}
```

## dapr operator server

dapr operator提供两个作用：
1. 提供DaprController服务，用于向k8s api server订阅自己关心的资源；比如：component、subscription、或者configuration; 以及针对annotations中的自定义参数进行资源分配和资源部署; 比如前面说的业务pod的dapr service
2. 获取三类资源数据，包括：component、subscribe和configuration等

那么这部分主要是讲解第二点：components， subscription和configuration的资源获取。

在dapr runtime启动时，会获取所有注册到ETCD的CRDs相关资源，并对业务POD中的daprd runtime进行全局相关参数设置(http拦截器、分布式跟踪、metrics等)或者初始化与第三方软件服务的初始化连接(components-contrib)等等；

```golang
// NewAPIServer返回一个api server，这个最终就是通过k8s api server client去获取daprd runtime所关注的各个类型资源
func NewAPIServer(client client.Client) Server {
    // client 就是指k8s api server client
    return &apiServer{
        Client:     client,
        updateChan: make(chan *componentsapi.Component, 1),
    }
}

// Run 用于启动dapr runtime api server服务
//
// 该服务提供grpc 6500的端口服务, 这个是daprd runtime内部端口服务。
// daprd runtime初始化启动时，会通过该grpc端口服务获取来自k8s etcd存储的相关组件、订阅和配置数据
func (a *apiServer) Run(certChain *dapr_credentials.CertChain) {
    lis, err := net.Listen("tcp", fmt.Sprintf(":%v", serverPort))
    if err != nil {
        log.Fatal("error starting tcp listener: %s", err)
    }

    opts, err := dapr_credentials.GetServerOptions(certChain)
    if err != nil {
        log.Fatal("error creating gRPC options: %s", err)
    }
    s := grpc.NewServer(opts...)
    operatorv1pb.RegisterOperatorServer(s, a)

    log.Info("starting gRPC server")
    if err := s.Serve(lis); err != nil {
        log.Fatalf("gRPC server error: %v", err)
    }
}

// GetConfiguration 通过传入资源的namespace和name，从k8s api server获取configuration(dapr runtime的全局配置)
func (a *apiServer) GetConfiguration(ctx context.Context, in *operatorv1pb.GetConfigurationRequest) (*operatorv1pb.GetConfigurationResponse, error) {
    key := types.NamespacedName{Namespace: in.Namespace, Name: in.Name}
    var config configurationapi.Configuration
    // 通过k8s api server client 获取业务pod自己的配置configuration，或者全局configuration
    if err := a.Client.Get(ctx, key, &config); err != nil {
        return nil, errors.Wrap(err, "error getting configuration")
    }
    b, err := json.Marshal(&config)
    if err != nil {
        return nil, errors.Wrap(err, "error marshalling configuration")
    }
    // 返回给dapr runtime
    return &operatorv1pb.GetConfigurationResponse{
        Configuration: b,
    }, nil
}

// ListComponents 获取components列表, 这个是获取整个集群中的所有dapr components资源配置
func (a *apiServer) ListComponents(ctx context.Context, in *emptypb.Empty) (*operatorv1pb.ListComponentResponse, error) {
    var components componentsapi.ComponentList
    // 从k8s api server client获取所有的components组件列表
    //
    // 注意：这里是获取k8s集群内所有业务配置的使用components集合资源，也就是说
    // 如果pod A业务使用的redis，那这个redis也会在暂时在另一个pod B业务中会进行初始化和动态更新
    // 虽然这个pod B业务暂时不会使用该redis，并不代表未来不会使用，所以一个业务使用某个component，集群内的所有业务pod中的daprd runtime container都会加载该component。进行与第三方软件服务的连接初始化动作
    if err := a.Client.List(ctx, &components); err != nil {
        return nil, errors.Wrap(err, "error getting components")
    }
    // 把从k8s api server获取的components列表数据，转换成目标协议数据列表，并返回给主调方dapr runtime
    resp := &operatorv1pb.ListComponentResponse{
        Components: [][]byte{},
    }
    for i := range components.Items {
        c := components.Items[i] // Make a copy since we will refer to this as a reference in this loop.
        b, err := json.Marshal(&c)
        if err != nil {
            log.Warnf("error marshalling component: %s", err)
            continue
        }
        resp.Components = append(resp.Components, b)
    }
    return resp, nil
}

// ListSubscriptions 获取subscriptions列表
func (a *apiServer) ListSubscriptions(ctx context.Context, in *emptypb.Empty) (*operatorv1pb.ListSubscriptionsResponse, error) {
    var subs subscriptionsapi.SubscriptionList
    // 根据subscription资源类型，从k8s api server中获取SubscriptionList资源列表
    if err := a.Client.List(ctx, &subs); err != nil {
        return nil, errors.Wrap(err, "error getting subscriptions")
    }
    // 并把获取到的SubscriptionList资源数据，转换成目标协议数据Subscriptions，并返回给上游daprd runtime
    resp := &operatorv1pb.ListSubscriptionsResponse{
        Subscriptions: [][]byte{},
    }
    for i := range subs.Items {
        s := subs.Items[i] // Make a copy since we will refer to this as a reference in   this loop.
        b, err := json.Marshal(&s)
        if err != nil {
            log.Warnf("error marshalling subscription: %s", err)
            continue
        }
        resp.Subscriptions = append(resp.Subscriptions, b)
    }
    return resp, nil
}

// ComponentUpdate component更新
//
// 也就是说components-contrib第三方软件服务列表中的某些参数或者配置发生变更
// 或者，业务pod服务需要对第三方软件服务商进行切换，这时需要进行component资源对象变更，并更新到所有的
// daprd runtime
func (a *apiServer) ComponentUpdate(in *emptypb.Empty, srv operatorv1pb.Operator_ComponentUpdateServer) error {
    log.Info("sidecar connected for component updates")

    // 通过k8s Operator捕获关心的资源变更通知事件={新增，更新}
    //
    // NewOperator会通过componentInfomer增加关注的资源事件，并异步通过updateChan来异步通知本dapr runtime container，进行资源更新
    for c := range a.updateChan {
        go func(c *componentsapi.Component) {
            // 把component资源对象转换成目标协议数据，并通过grpc stream推送给上游dapr runtime server
            b, err := json.Marshal(&c)
            if err != nil {
                log.Warnf("error serializing component %s (%s): %s", c.GetName(), c.Spe
c.Type, err)
                return
            }
            err = srv.Send(&operatorv1pb.ComponentUpdateEvent{
                Component: b,
            })
            if err != nil {
                log.Warnf("error updating sidecar with component %s (%s): %s", c.GetNam
e(), c.Spec.Type, err)
                return
            }
            log.Infof("updated sidecar with component %s (%s)", c.GetName(), c.Spec.Typ
e)
        }(c)
    }
    return nil
}
```

通过上面k8s dapr operator server提供的components、subscriptions和configuration三类CRDs资源配置，当业务pod的dapr runtime container进行初始化时，会通过k8s dapr operator提供的3500 grpc服务来获取这三类CRDs资源对象，当有component发生变更时，也会进行整个k8s使用了dapr注入的业务pod进行component更新。

小结，这前面k8s operator两部分讲述了k8s DaprController注册和订阅关心的资源对象，并进行业务pod的dapr service部署和删除工作；以及为dapr runtime主服务在初始化阶段提供components、subscriptions和configuration资源对象的获取，并对这些资源对象进行初始化工作。当k8s componentInfomer收到component资源对象的变更时(新增、更新)，就会通知业务pod的dapr runtime，并进行初始化或者重连动作。


### dapr operator server

初始化和启动dapr operator服务，并为dapr runtime提供grpc服务，获取三类CRDs资源对象。

```golang
// NewOperator 在dapr-system空间下创建dapr-operator pod服务实例
func NewOperator(config, certChainPath string, enableLeaderElection bool) Operator {
    // 通过~/.kube/config.yaml配置文件初始化与k8s api server的连接，并获得manager
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:             scheme,
        MetricsBindAddress: "0",
        LeaderElection:     enableLeaderElection,
        LeaderElectionID:   "operator.dapr.io",
    })
    if err != nil {
        log.Fatal("unable to start manager")
    }
    // 通过manager获取k8s api server的client以及注册CRDs的scheme对象
    daprHandler := handlers.NewDaprHandler(mgr)
    // 把k8s operator所有关注的资源对象，包括Deployment、Service等资源对象生命周期联系起来。统一管理
    if err := daprHandler.Init(); err != nil {
        log.Fatalf("unable to initialize handler, err: %s", err)
    }

    // 构建k8s operator 对象，并在dapr-system命名空间下为所有业务pod的dapr runtime container提供grpc服务，用于获取业务pod注册各类CRDs资源对象。包括subscriptions、components和configuration资源对象
    o := &operator{
        daprHandler:   daprHandler,
        mgr:           mgr,
        client:        mgr.GetClient(),
        configName:    config,
        certChainPath: certChainPath,
    }
    o.apiServer = api.NewAPIServer(o.client)
    // 在把会发生动态变更(新增、更新)的component资源对象，进行订阅并通知给各个业务dapr runtime运行时
    // 这里是通过syncComponent的updateChan异步方式来给到ComponentUpdate API最终通过grpc stream推送给
    // 各个业务pod的dapr runtime container服务
    //
    // 增加订阅通知事件
    if componentInfomer, err := mgr.GetCache().GetInformer(context.TODO(), &componentsapi.Component{}); err != nil {
        log.Fatalf("unable to get setup components informer, err: %s", err)
    } else {
        componentInfomer.AddEventHandler(cache.ResourceEventHandlerFuncs{
            AddFunc: o.syncComponent,
            UpdateFunc: func(_, newObj interface{}) {
                o.syncComponent(newObj)
            },
        })
    }
    return o
}

// prepareConfig 用于获取dapr-system命名空间的k8s dapr operator服务配置
func (o *operator) prepareConfig() {
    var err error
    o.config, err = LoadConfiguration(o.configName, o.client)
    if err != nil {
        log.Fatalf("unable to load configuration, config: %s, err: %s", o.configName, e  rr)
    }
    o.config.Credentials = credentials.NewTLSCredentials(o.certChainPath)
}

// syncComponent 通过componentInfomer订阅component资源事件，来推送给所有的业务pod dapr runtime
func (o *operator) syncComponent(obj interface{}) {
    c, ok := obj.(*componentsapi.Component)
    if ok {
        log.Debugf("observed component to be synced, %s/%s", c.Namespace, c.Name)
        o.apiServer.OnComponentUpdated(c)
    }
}

// Run 在dapr-system命名空间下启动dapr-operator pod服务
//
// 启动k8s dapr operator和对所有业务pod提供的grpc 3500端口服务(获取subscription、component和configuration CRDs资源对象)
func (o *operator) Run(ctx context.Context) {
    defer runtimeutil.HandleCrash()
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    o.ctx = ctx
    go func() {
        <-ctx.Done()
        log.Infof("Dapr Operator is shutting down")
    }()
    log.Infof("Dapr Operator is started")

    go func() {
        // 启动k8s dapr operator,也就是DaprController控制器，订阅所关注的资源对象
        if err := o.mgr.Start(ctx); err != nil {
            if err != nil {
                log.Fatalf("failed to start controller manager, err: %s", err)
            }
        }
    }()
    if !o.mgr.GetCache().WaitForCacheSync(ctx) {
        log.Fatalf("failed to wait for cache sync")
    }
    o.prepareConfig()

    var certChain *credentials.CertChain
    if o.config.MTLSEnabled {
        log.Info("mTLS enabled, getting tls certificates")
        // try to load certs from disk, if not yet there, start a watch on the local fi  lesystem
        chain, err := credentials.LoadFromDisk(o.config.Credentials.RootCertPath(), o.c  onfig.Credentials.CertPath(), o.config.Credentials.KeyPath())
        if err != nil {
            fsevent := make(chan struct{})

            go func() {
                log.Infof("starting watch for certs on filesystem: %s", o.config.Creden  tials.Path())
                err = fswatcher.Watch(ctx, o.config.Credentials.Path(), fsevent)
                if err != nil {
                    log.Fatal("error starting watch on filesystem: %s", err)
                }
            }()

            <-fsevent
            log.Info("certificates detected")

            chain, err = credentials.LoadFromDisk(o.config.Credentials.RootCertPath(),   o.config.Credentials.CertPath(), o.config.Credentials.KeyPath())
            if err != nil {
                log.Fatal("failed to load cert chain from disk: %s", err)
            }
        }
        certChain = chain
        log.Info("tls certificates loaded successfully")
    }

    go func() {
        // 启动dapr-system命名空间下的k8s dapr operator服务的健康检查
        healthzServer := health.NewServer(log)
        healthzServer.Ready()

        err := healthzServer.Run(ctx, healthzPort)
        if err != nil {
            log.Fatalf("failed to start healthz server: %s", err)
        }
    }()

    // 启动所有业务pod的dapr runtime sidecar提供的grpc 3500端口服务，去获取三类CRDs资源或者主动推送component资源
    o.apiServer.Run(certChain)
}
```

小结：通过上面k8s dapr operator实例的创建，并通过dapr-system命名空间下dapr-operator服务的启动，会创建dapr operator服务，以及对所有业务pod提供grpc 3500端口服务，用于获取CRDs资源对象。并根据这些CRDs资源对象，进行dapr runtime的接收数据处理(拦截器、metrics、分布式跟踪)。以及第三方软件服务components-contrib的初始化连接操作。


### k8s operator client

k8s operator client并不是dapr-system下dapr-operator pod服务提供的端口服务，而是通过这个operator/client包，可以建立与dapr-operator系统服务的grpc client服务连接。这样业务pod的dapr runtime sidecar就可以获取grpc 3500端口服务的client，并获取configuration、subscription和component三类CRDs资源对象

```golang
// GetOperatorClient 通过address和serviceName，建立并获取与grpc 3500端口服务的通信client
// 这样业务pod中的dapr runtime sidecar就可以与k8s dapr operator进行通信，获取CRDs资源
func GetOperatorClient(address, serverName string, certChain *dapr_credentials.CertChain) (operatorv1pb.OperatorClient, *grpc.ClientConn, error) {
    unaryClientInterceptor := grpc_retry.UnaryClientInterceptor()

    if diag.DefaultGRPCMonitoring.IsEnabled() {
        unaryClientInterceptor = grpc_middleware.ChainUnaryClient(
            unaryClientInterceptor,
            diag.DefaultGRPCMonitoring.UnaryClientInterceptor(),
        )
    }

    opts := []grpc.DialOption{grpc.WithUnaryInterceptor(unaryClientInterceptor)}

    if certChain != nil {
        cp := x509.NewCertPool()
        ok := cp.AppendCertsFromPEM(certChain.RootCA)
        if !ok {
            return nil, nil, errors.New("failed to append PEM root cert to x509 CertPoo  l")
        }

        config, err := dapr_credentials.TLSConfigFromCertAndKey(certChain.Cert, certCha  in.Key, serverName, cp)
        if err != nil {
            return nil, nil, errors.Wrap(err, "failed to create tls config from cert an  d key")
        }
        opts = append(opts, grpc.WithTransportCredentials(credentials.NewTLS(config)))
    } else {
        opts = append(opts, grpc.WithInsecure())
    }

    conn, err := grpc.Dial(address, opts...)
    if err != nil {
        return nil, nil, err
    }
    return operatorv1pb.NewOperatorClient(conn), conn, nil
}
```


### 总结

通过k8s dapr operator创建DaprController控制器注册和订阅所关心的资源对象,Deployment， Service和配置中的annotations dapr自定义字段值，去动态地为业务pod创建dapr service，所以业务pod的主container并没有直接通过自身的端口服务用service暴露出去，而是暴露的业务pod中dapr runtime sidecar中的各个端口服务作为service，供外部使用；

这个业务pod中的dapr service提供了http 80端口服务、grpc 50001端口服务， grpc 50002内部端口服务以及metrics的9090端口服务

同时dap-system命名空间下的这个dapr-operator系统服务，也对所有业务pod提供了获取CRDs三类资源(components，subscriptions和configuration资源)的服务，使得业务pod中的dapr runtime sidecar在初始化时进行消息订阅、拦截器、metrics指标上报、证书建立等相关连接初始化的动作。并且在业务pod启用或者变更component资源时，进行初始化或者重连与第三方软件服务的通信动作。这样后续其他业务pod可以直接复用这些系统已有的资源服务。
