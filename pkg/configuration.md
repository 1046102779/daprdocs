目前还剩下apis、config、grpc、messages、channel、messaging、client、http、metrics、proto、components、diagnostics和middleware，以及testing包还没有进行源码阅读

credentials在dapr sentry有引用
messages在dapr messages包中表示errors列表，显示目前可以使用的错误变量列表

## config

config是对dapr configuration yaml配置的解析和参数校验, 完整的configuration yaml配置文件所有参数如下所示:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: secretappconfig
spec:
  secrets:
    scopes:
        - storeName: "local"
          defaultAccess: "allow"
          allowedSecrets: ["daprsecret","redissecret"]
          deniedSecrets: ["xxx", "xxx"]
  metric:
    enabled: false
  tracing:
    samplingRate: 0.8
    stdout: true
    zipkin:
        endpointAddress: "127.0.0.1:XXXX"
  mtls:
    allowedClockSkew: "xx"
    enabled: true
    workloadCertTTL: "xxx"
  httpPipeline:
    handlers:
      - name: "xxx"
        selector:
          fields:
            - field: "xxx"
              value: "xxx"
        type: "xxx"
        version: "xxx"
  accessControl:
    defaultAction: "xx"
    policies:
      defaultAction: "xxx"  
      policies:
        - appId: "xxx"
          defaultAction: "xxx"
          namespace: "xxx"
          trustDomain: "xxx"
          operations:
            - action: "xxx"
              httpVerb: ["xxx", "xxx"]
              name: "xxx"
      trustDomain: "xxx"
```

针对上面这个完整的配置文件, 进行解析并构建成内存对象，接下来，就针对这个yaml文件进行解析

```golang
// LoadDefaultConfiguration 加载创建一个Configuration实例对象
func LoadDefaultConfiguration() *Configuration {
    // 它主要是对configuration yaml CRDs文件的spec数据部分进行初始化创建
    // 包括：tracing, metric和accessControl
    // 而并没有对mtls, serets两个数据部分进行初始化
    return &Configuration{
        Spec: ConfigurationSpec{
            // tracing，是指zipkin分布式跟踪采样率和采集服务端口地址
            TracingSpec: TracingSpec{
                SamplingRate: "",
            },
            // metric 指标采集是否开启
            MetricSpec: MetricSpec{
                Enabled: true,
            },
            // accessControl 访问控制，用于内部dapr runtime之间的访问控制，并与dapr sentry中的spiffed标准结合
            // 目前所有的访问或者secrets，是否可以通过，由两个值控制，一个allow，一个deny。再无其他
            AccessControlSpec: AccessControlSpec{
                DefaultAction: AllowAccess,
                TrustDomain:   "public",
            },
        },
    }
}

// LoadStandaloneConfiguration 加载dapr configuration配置文件, 并对参数进行校验和排序
func LoadStandaloneConfiguration(config string) (*Configuration, string, error) {
    // 校验config path configuration.yaml文件是否存在
    _, err := os.Stat(config)
    if err != nil {
        return nil, "", err
    }

    // 读取config文件数据流
    b, err := ioutil.ReadFile(config)
    if err != nil {
        return nil, "", err
    }

    // 通过LoadDefaultConfiguration方法创建一个Configuration空对象
    conf := LoadDefaultConfiguration()
    // 在把config文件内容反序列化到configuration对象中
    err = yaml.Unmarshal(b, conf)
    if err != nil {
        return nil, string(b), err
    }
    // 并对configuration.yaml文件内容进行校验，主要就是spec数据部分内容校验
    err = sortAndValidateSecretsConfiguration(conf)
    if err != nil {
        return nil, string(b), err
    }

    // 返回
    return conf, string(b), nil
}

// sortAndValidateSecretsConfiguration 方法用于对configuration配置对象的secrets部分数据校验
  func sortAndValidateSecretsConfiguration(conf *Configuration) error {
    // 目前只针对spec部分的serets字段校验
    scopes := conf.Spec.Secrets.Scopes
    // set类似于map[string]struct{}，集合校验key是否存在
    set := sets.NewString()
    // 遍历secrets中的每个scope，并对旗下的数据进行校验
    for _, scope := range scopes {
        // 验证scope的storename是否已经存在，如果已存在，说明存在冲突报错
        if set.Has(scope.StoreName) {
            return errors.Errorf("%q storeName is repeated in secrets configuration", s  cope.StoreName)
        }
        // 校验scope下默认访问权限如果不为空，那么DefaultAccess必须是allow或者deny二者其一
        // 否则，报错
        if scope.DefaultAccess != "" &&
            !strings.EqualFold(scope.DefaultAccess, AllowAccess) &&
            !strings.EqualFold(scope.DefaultAccess, DenyAccess) {
            return errors.Errorf("defaultAccess %q can be either allow or deny", scope.  DefaultAccess)
        }
        // 临时storename写入到内存中
        set.Insert(scope.StoreName)

        // 并对scope下的allowedsecrets和deniedsecrets分别排序
        // 这个接下来会用于二分法查找
        sort.Strings(scope.AllowedSecrets)
        sort.Strings(scope.DeniedSecrets)
    }

    return nil
}

// LoadKubernetesConfiguration 用于从k8s获取configuration CRDs对象数据
func LoadKubernetesConfiguration(config, namespace string, operatorClient operatorv1pb.  OperatorClient) (*Configuration, error) {
    // 在k8s dapr-system命名空间下，dapr-operator服务对外提供了grpc服务，dapr runtime可以通过GetConfiguration方法获取指定namespace和name的configuration配置对象
    resp, err := operatorClient.GetConfiguration(context.Background(), &operatorv1pb.Ge  tConfigurationRequest{
        Name:      config,
        Namespace: namespace,
    }, grpc_retry.WithMax(operatorMaxRetries), grpc_retry.WithPerRetryTimeout(operatorC  allTimeout))
    if err != nil {
        return nil, err
    }
    if resp.GetConfiguration() == nil {
        return nil, errors.Errorf("configuration %s not found", config)
    }
    // 创建一个空对象Configuration，并通过yaml反序列化把grpc 得到的数据流解析到conf对象中
    conf := LoadDefaultConfiguration()
    err = json.Unmarshal(resp.GetConfiguration(), conf)
    if err != nil {
        return nil, err
    }

    // 并对spec部分的serets数据对象进行校验和验证
    err = sortAndValidateSecretsConfiguration(conf)
    if err != nil {
        return nil, err
    }

    return conf, nil
}

// IsSecretAllowed 校验key值在secret下的scope中是否被允许
func (c SecretsScope) IsSecretAllowed(key string) bool {
    // 如果scope下的allowedserets包含key，则表示storename允许该key通过
    if len(c.AllowedSecrets) != 0 {
        return containsKey(c.AllowedSecrets, key)
    }

    // 如果scope下的deniedserets包含可以，则表示storename拒绝该key通过
    if deny := containsKey(c.DeniedSecrets, key); deny {
        return !deny
    }

    // 如果该key既不属于allow，也不属于deny，则看看scope的默认访问测试是否为拒绝
    // 如果是拒绝，表示该key不允许通过
    // 否则，允许该key通过。
    return !strings.EqualFold(c.DefaultAccess, DenyAccess)
}

// containsKey 因为scope下的AllowedSerets和DeniedSerets已经是排序OK的，则通过二分法查找key是否存在
func containsKey(s []string, key string) bool {
    // 二分法超着key是否存在，并返回key所在的索引位置
    // 如果key不存在，则返回要插入的位置
    index := sort.SearchStrings(s, key)

    // 如果返回的索引小于s长度，再看索引位置是否等于key，如果是表示在s列表中存在key。
    return index < len(s) && s[index] == key
}

***************************************************
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: allowlistsappconfig
spec:
  accessControl:
    defaultAction: deny
    trustDomain: "public"
    policies:
    - appId: "allowlists-caller"
      defaultAction: deny
      trustDomain: 'public'
      namespace: "dapr-tests"
      operations:
      - name: /opAllow
        httpVerb: ['POST', 'GET']
        action: allow
      - name: /opDeny
        httpVerb: ["*"]
        action: deny
***************************************************

// ParseAccessControlSpec 解析.spec.accesscontrol部分数据, 并校验数据是否合理
func ParseAccessControlSpec(accessControlSpec AccessControlSpec, protocol string) (*Acc  essControlList, error) {
    // accessControl是一个dapr runtime之间的RPC访问控制策略，它通过配置和spiffed标准相结合来实现内部访问控制.

    // 如果accessControl下的信任域, 默认为public，默认命名空间为default
    // 如果trustDomain、defaultAction和appPolicies都为空，则accessControl为无效配置
    if accessControlSpec.TrustDomain == "" &&
        accessControlSpec.DefaultAction == "" &&
        len(accessControlSpec.AppPolicies) == 0 {
        // No ACL has been specified
        log.Debugf("No Access control policy specified")
        return nil, nil
    }

    // 这里我们把accessControl配置对象转换成AccessControlList内存对象
    var accessControlList AccessControlList
    accessControlList.PolicySpec = make(map[string]AccessControlListPolicySpec)

    // 如果配置对象的信任域为空，则为默认信任域public
    accessControlList.TrustDomain = accessControlSpec.TrustDomain
    if accessControlSpec.TrustDomain == "" {
        accessControlList.TrustDomain = DefaultTrustDomain
    }

    // 目前配置对象的action有两种，一种为allowed，一种为deny。
    accessControlList.DefaultAction = strings.ToLower(accessControlSpec.DefaultAction)
    // 如果默认动作为空，且apppolicies长度大于0，则accessControl的action为deny
    if accessControlSpec.DefaultAction == "" {
        if len(accessControlSpec.AppPolicies) > 0 {
            // Some app level policies have been specified but not default global actio  n is set. Default to more secure option - Deny
            log.Warnf("No global default action has been specified. Setting default glo  bal action as Deny")
            accessControlList.DefaultAction = DenyAccess
        } else {
            // 如果默认动作为空，且policies长度也为0，则表示默认允许通过
            accessControlList.DefaultAction = AllowAccess
        }
    }

    var (
        invalidTrustDomain []string
        invalidNamespace   []string
        invalidAppName     bool
    )
    // 再对accesscontrol下一层appPolicies对象进行校验和对象转换
    accessControlList.PolicySpec = make(map[string]AccessControlListPolicySpec)
    for _, appPolicySpec := range accessControlSpec.AppPolicies {
        // 如果appPolicies的appid为空，则需要统计appid的错误
        if appPolicySpec.AppName == "" {
            invalidAppName = true
        }

        // 如果appPolicies的信任域为空，也需要统计该错误，统计为appid的类别错误
        invalid := false
        if appPolicySpec.TrustDomain == "" {
            invalidTrustDomain = append(invalidTrustDomain, appPolicySpec.AppName)
            invalid = true
        }
        // 如果appPolices的命名空间为空，也需要统计该错误，统计为appid的类别错误
        if appPolicySpec.Namespace == "" {
            invalidNamespace = append(invalidNamespace, appPolicySpec.AppName)
            invalid = true
        }

        // 这个用于汇总三类错误：appId为空，信任域和命名空间为空；
        // 这里是通过appPolicies列表的所有汇总错误
        if invalid || invalidAppName {
            // An invalid config was found for this app. No need to continue parsing th  e spec for this app
            continue
        }

        // 运行到这里，表示appPolicies列表下的appId、信任域和namespace参数没有错误
        operationPolicy := make(map[string]AccessControlListOperationAction)

        // 然后再针对每个appPolicy下的actions进行对象转换
        for _, appPolicy := range appPolicySpec.AppOperationActions {
            // appPolicy={name, httpVerb, action}, 如
            /*
              - name: /opAllow
                httpVerb: ["POST", "GET"]
                action: allow
            */
            // 首先把name值转换成/opAllow和/, 前者为operationPrefix，后者为operationPostfix
            operation := appPolicy.Operation
            if !strings.HasPrefix(operation, "/") {
              operation = "/" + operation
            }
            operationPrefix, operationPrefix := getOperationPrefixAndPostfix(operation  )

            // 如果为http协议，则把operationPrefix和operationPrefix转换为小写
            if protocol == HTTPProtocol {
                operationPrefix = strings.ToLower(operationPrefix)
                operationPostfix = strings.ToLower(operationPostfix)
            }

            // 把路由和动作放在operationPrefix下
            operationActions := AccessControlListOperationAction{
                OperationPostFix: operationPostfix,
                VerbAction:       make(map[string]string),
            }

            // 针对每个请求动作，比如POST或者GET请求动作，存储动作的对应策略
            // 比如GET，允许；POST，允许等
            for _, verb := range appPolicy.HTTPVerb {
                operationActions.VerbAction[verb] = appPolicy.Action
            }

            // 并把动作和对应的策略存储到OperationAction中
            operationActions.OperationAction = appPolicy.Action

            // 最后在统一存储在appPolicy的前缀路由下
            operationPolicy[operationPrefix] = operationActions
        }
        // 然后在把appPolicies列表转换amp后的数据对象，存储到appPolicy
        aclPolicySpec := AccessControlListPolicySpec{
            AppName:             appPolicySpec.AppName,
            DefaultAction:       appPolicySpec.DefaultAction,
            TrustDomain:         appPolicySpec.TrustDomain,
            Namespace:           appPolicySpec.Namespace,
            AppOperationActions: operationPolicy,
        }

        // 并把appId||namespace作为key，存储到accessControlList的policy策略中
        // 这样任何一个dapr runtime之间的RPC访问，都可以通过该key查看某个资源是否具有访问权限
        key := getKeyForAppID(aclPolicySpec.AppName, aclPolicySpec.Namespace)
        accessControlList.PolicySpec[key] = aclPolicySpec
    }

    // 如果accessControl对象中的信任域为空、或者无效appid，又或者无效命名空间，则返回错误
    if len(invalidTrustDomain) > 0 || len(invalidNamespace) > 0 || invalidAppName {
        return nil, errors.Errorf(
            "invalid access control spec. missing trustdomain for apps: %v, missing nam  espace for apps: %v, missing app name on at least one of the app policies: %v",
            invalidTrustDomain,
            invalidNamespace,
            invalidAppName)
    }

    // 最终我们把accessControl配置对象，转换成了accesssControllList。它将来会用于内部dapr runtime之间的内部RPC调用访问权限控制, 资源控制粒度细。
    return &accessControlList, nil
}

// GetAndParseSpiffeID 通过grpc context上下文获取证书，并从证书中获取spiffe，这个在dapr sentry源码阅读中已经说明过, 然后在对spiffeID进行反序列化为SpifffID
func GetAndParseSpiffeID(ctx context.Context) (*SpiffeID, error) {
    // 首先从grpc context的证书扩展字段中获取spiffed内容
    spiffeID, err := getSpiffeID(ctx)
    if err != nil {
        return nil, err
    }

    // spiffeID格式如下：
    // spiffe://trustDomain/ns/namespace/appID
    // 然后再把spiffeID字符串转换为标准的SpiffeID
    id, err := parseSpiffeID(spiffeID)
    return id, err
}

// parseSpiffeID  把spiffe://trustDomain/ns/namespace/appID转换为SpiffeID
func parseSpiffeID(spiffeID string) (*SpiffeID, error) {
    // 如果spiffeID为空，则直接返回错误
    if spiffeID == "" {
        return nil, errors.New("input spiffe id string is empty")
    }

    // 校验spiffeID是否以spiffe://开头
    if !strings.HasPrefix(spiffeID, SpiffeIDPrefix) {
        return nil, errors.New(fmt.Sprintf("input spiffe id: %s is invalid", spiffeID))
    }

    // The SPIFFE Id will be of the format: spiffe://<trust-domain/ns/<namespace>/<app-id>
    parts := strings.Split(spiffeID, "/")
    if len(parts) < 6 {
        return nil, errors.New(fmt.Sprintf("input spiffe id: %s is invalid", spiffeID))
    }

    // 解析spiffeID，并填充到SpiffeID对象中返回
    var id SpiffeID
    // 获取TrustDomain、Namespace和AppID，表示该证书允许访问的dapr runtime服务
    id.TrustDomain = parts[2]
    id.Namespace = parts[4]
    id.AppID = parts[5]

    return &id, nil
}

// getSpiffeID 从grpc context上下文的证书扩展字段中获取spiffe数据
func getSpiffeID(ctx context.Context) (string, error) {
    var spiffeID string
    // p, ok = ctx.Value(peerKey{}).(*Peer) 获取Peer
    peer, ok := peer.FromContext(ctx)
    if ok {
        if peer == nil || peer.AuthInfo == nil {
            return "", errors.New("unable to retrieve peer auth info")
        }

        // 断言是否为credentials.TLSInfo
        tlsInfo := peer.AuthInfo.(credentials.TLSInfo)

        // https://www.ietf.org/rfc/rfc3280.txt
        oid := asn1.ObjectIdentifier{2, 5, 29, 17}

        // 查找每个证书中扩展字段列表中每个扩展对象ID，与标准的spiffe oid相等的扩展字段extension
        for _, crt := range tlsInfo.State.PeerCertificates {
            for _, ext := range crt.Extensions {
                // 如果extension的ID与OID相等，则表示该extension存储了spiffe信息
                if ext.Id.Equal(oid) {
                    // 通过asn1标准协议解析extension的value值
                    var sequence asn1.RawValue
                    if rest, err := asn1.Unmarshal(ext.Value, &sequence); err != nil {
                        log.Debug(err)
                        continue
                    } else if len(rest) != 0 {
                        log.Debug("the SAN extension is incorrectly encoded")
                        continue
                    }

                    if !sequence.IsCompound || sequence.Tag != asn1.TagSequence || sequ  ence.Class != asn1.ClassUniversal {
                        log.Debug("the SAN extension is incorrectly encoded")
                        continue
                    }

                    // 把解析到的数据流转换成字符串，如果字符串前缀为spiffe://
                    // 则表示grpc context上下文中从证书列表中找到了spiffe
                    for bytes := sequence.Bytes; len(bytes) > 0; {
                        var rawValue asn1.RawValue
                        var err error

                        bytes, err = asn1.Unmarshal(bytes, &rawValue)
                        if err != nil {
                            return "", err
                        }

                        spiffeID = string(rawValue.Bytes)
                        if strings.HasPrefix(spiffeID, SpiffeIDPrefix) {
                            return spiffeID, nil
                        }
                    }
                }
            }
        }
    }

    return "", nil
}

// IsOperationAllowedByAccessControlPolicy 校验该请求是否可以访问指定的dapr runtime服务
func IsOperationAllowedByAccessControlPolicy(spiffeID *SpiffeID, srcAppID string, inputOperation string, httpVerb common.HTTPExtension_Verb, appProtocol string, accessControlList *AccessControlList) (bool, string) {
    // 如果accessControlList为空，表示访问不限制，直接返回true
    if accessControlList == nil {
        // No access control list is provided. Do nothing
        return isActionAllowed(AllowAccess), ""
    }

    // 否则，表示accessControlList不为空，则需要进行访问权限控制查询

    // accessControllList的总默认动作两种：一个allow，一个为deny
    action := accessControlList.DefaultAction
    // 动作策略是全局的
    actionPolicy := ActionPolicyGlobal

    // 如果主调appid为空，表示执行默认的策略，如果accessControlList默认动作为deny，则表示不允许访问目标appid
    if srcAppID == "" {
        // Did not receive the src app id correctly
        return isActionAllowed(action), actionPolicy
    }

    // 如果spiffeID为空，表示主调证书空，则执行默认策略，如果accessControlList默认动作为deny，则表示不允许访问目标appid
    if spiffeID == nil {
        // Could not retrieve spiffe id or it is invalid. Apply global default action
        return isActionAllowed(action), actionPolicy
    }

    // 再构建形成appID||namespace, 作为key，来校验accessControlList下policy策略是否存在该key
    // 如果存在，则说明有针对该key的ACL权限访问控制限制
    key := getKeyForAppID(srcAppID, spiffeID.Namespace)
    appPolicy, found := accessControlList.PolicySpec[key]

    // 没有发现，则执行默认的访问动作，如果accessControlList的defaultAction为deny，则不允许该请求访问appid
    if !found {
        // no policies found for this src app id. Apply global default action
        return isActionAllowed(action), actionPolicy
    }

    // 匹配主调的信任域和ACL的访问信任域，不相同，则执行默认的动作策略
    if appPolicy.TrustDomain != spiffeID.TrustDomain {
        return isActionAllowed(action), actionPolicy
    }

    // 匹配主调的namespace与ACL控制的namespace，不相同则执行默认的动作策略
    if appPolicy.Namespace != spiffeID.Namespace {
        return isActionAllowed(action), actionPolicy
    }

    // 如果前面通过，则在进行appPolicy级别的ACL权限控制访问
    if appPolicy.DefaultAction != "" {
        // appPolicy默认策略动作为DefaultAction
        action = appPolicy.DefaultAction
        // 动作策略是appPolicy级别
        actionPolicy = ActionPolicyApp
    }

    // 添加路由前缀"/", 并获取prefix, postfix
    // “/v1/users/” prefix=/v1 postfix=/users
    if !strings.HasPrefix(inputOperation, "/") {
        inputOperation = "/" + inputOperation
    }
    inputOperationPrefix, inputOperationPostfix := getOperationPrefixAndPostfix(inputOp  eration)

    // 如果为http请求，则把路由改为小写
    if appProtocol == HTTPProtocol {
        inputOperationPrefix = strings.ToLower(inputOperationPrefix)
        inputOperationPostfix = strings.ToLower(inputOperationPostfix)
    }

    // 进行appPolicy的路由前缀匹配，如果找到，则在进行动作和请求方法匹配
    operationPolicy, found := appPolicy.AppOperationActions[inputOperationPrefix]
    if found {
       // 路由匹配， 如果后缀路由以/*开头，则需要把/*替换为"",然后再进行后缀的前缀匹配
       // 如果匹配失败，则返回appPolicy级别的默认策略
       if strings.Contains(operationPolicy.OperationPostFix, "/*") {
           if !strings.HasPrefix(inputOperationPostfix, strings.ReplaceAll(operationPo  licy.OperationPostFix, "/*", "")) {
               return isActionAllowed(action), actionPolicy
           }
       } else {
           // r如果不是以/*后缀结尾，则直接比较后缀路由，如果不相同则直接返回appPolicy级别的动作策略
           if operationPolicy.OperationPostFix != inputOperationPostfix {
               return isActionAllowed(action), actionPolicy
           }
       }

       // 就如果协议为http，则校验访问的http 访问，比如get，post等，返回该方法是否允许访问
       if appProtocol == HTTPProtocol {
           if httpVerb != common.HTTPExtension_NONE {
               verbAction, found := operationPolicy.VerbAction[httpVerb.String()]
               if found {
                   // 发现了该访问，再把该方法的执行策略赋值给action
                   action = verbAction
               } else {
                  // 如果不存在，则校验是否具有*，则针对所有方法的访问权限
                   verbAction, found = operationPolicy.VerbAction["*"]
                   if found {
                       // The verb matched the wildcard "*"
                       action = verbAction
                   }
               }
           } else {
               // 否则执行appPolicy的默认动作策略
               action = appPolicy.DefaultAction
           }
       } else if appProtocol == GRPCProtocol {
           // 如果协议为grpc，则只需要赋值该appPolicy下的action动作策略，不需要进行请求方法的匹配，比如GET、POST等
           action = operationPolicy.OperationAction
       }
   }

   // 否则执行默认策略
   return isActionAllowed(action), actionPolicy
}
```


## 总结

通过上面的分析，我们可以知道整个dapr configuration配置，是如何加载的，如何解析，以及configuration支持httpPipeline、metric、tracing、mtls、accessControl五种能力，并针对spiffeID标准来实现微服务之前访问的ACL访问权限细粒度控制，并针对主调与被调RPC访问的权限控制梳理。正是通过accessControl实现RPC内部访问的保护，防止内部服务被攻击。
