
本文对dapr的validation、modes、scopes、version、signals和cors六个包进行源码分析

## modes

目前dapr的部署支持两种模式，一种是k8s；另一种是standalone(host)。

## validation

该dapr validation库主要是针对业务POD的app-id设置，进行命名规范来符合k8s资源定义名称的设置。

因为在业务POD中设置dapr.io/app-id时，k8s只会对资源的实例名称进行名称校验，所以annotations是不会校验的，这时候就需要在Daprcontroller订阅到该资源时，去annotations中的数据进行k8s命名规范校验，然后构建生成service dapr app-id或者其他需要以app-id来创建资源的命名规范，包括component、configuration等等CRDs。

那么命名规范只需要保持与k8s的命名规范规则一样就OK了。所以这里需要借助k8s的命名规范，把相关参数值校验方法拿来即用就行。

[k8s validation](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/validation/validation.go)

主要采用正则表达式对appid进行校验，表达式: **^[a-z0-9]([-a-z0-9]*[a-z0-9])?$**， 校验规则的方法：**isDNS1123Label**

然后再通过一个封装方法，来达到对appid进行规则校验的目的：

```golang
// ValidateKubernetesAppID 对appid进行k8s资源命名规则校验
func ValidateKubernetesAppID(appID string) error {
    // appID 不能为空
    if appID == "" {
        return errors.New("value for the dapr.io/app-id annotation is empty")
    }
    // 正则表达式匹配appid, 是否符合k8s规范
    r := isDNS1123Label(appID)
    if len(r) == 0 {
        return nil
    }
    s := fmt.Sprintf("invalid app id(input: %s): %s", appID, strings.Join(r, ","))
    return errors.New(s)
}
```

## version

返回version和commit。

1. version，表示当前代码库所在的版本号，当没有发布1.0.0的版本之前，全部都是edge；
2. commit。表示CI时构建服务时，所用的版本commitid。

## scopes

这个dapr scopes是对pubsub的发布和订阅功能进行范围控制定义。

因为可能一个资源配置文件CRDs，可以对整个dapr集群的所有appid进行资源使用配置，如下所示:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: default
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: "localhost:6379"
  - name: redisPassword
    value: ""
  - name: publishingScopes
    value: "app1=topic1;app2=topic2,topic3;app3="
  - name: subscriptionScopes
    value: "app2=;app3=topic1"
```

The table below shows which applications are allowed to publish into the topics:

|      | topic1 | topic2 | topic3 |
|------|--------|--------|--------|
| app1 | X      |        |        |
| app2 |        | X      | X      |
| app3 |        |        |        |

The table below shows which applications are allowed to subscribe to the topics:

|      | topic1 | topic2 | topic3 |
|------|--------|--------|--------|
| app1 | X      | X      | X      |
| app2 |        |        |        |
| app3 | X      |        |        |

上面这个配置是3个服务，三个topic。，我们可以看到app1可以通过配置publishingScopes，来达到发布topic消息的能力；然后app1也可以通过subscriptionScopes来达到订阅topic的能力。

注意，如果不进行显示设置，则默认为全部发布和订阅topic。如：app1的订阅，以及app2的不订阅能力

接下来主要对这个CRDs资源文件进行解析, 来校验指定appid是否具有订阅和发布topic的能力


```golang
const (
    // dapr的pubsub发布和订阅能力，归属于component CRDs资源配置中
    // SubscriptionScopes表示资源订阅范围： topic和appid列表
    SubscriptionScopes = "subscriptionScopes"
    // PublishingScopes 表示资源发布的范围：topic和appid列表
    PublishingScopes   = "publishingScopes"
    // AllowedTopics 表示资源允许的topic列表
    AllowedTopics      = "allowedTopics"
    // 解析topic与appid的资源分隔符种类
    appsSeperator      = ";"
    appSeperator       = "="
    topicSeperator     = ","
)

// GetScopedTopics 根据scope key值，返回指定appID订阅的topic列表
func GetScopedTopics(scope, appID string, metadata map[string]string) []string {
    topics := []string{}

    // 校验metadata中是否存在scope范围, 如果不存在，则表示没有设置过scope，意味着没有topic和appid使用scope能力
    //
    // 否则，表示scope存在，则继续查找appID订阅的topic列表
    if val, ok := metadata[scope]; ok && val != "" {
        // val如果为空，表示全部appid，则是返回全部的topic，还是返回空topic呢？
        // 这是个需要考虑的问题，后面实验下 ::TODO
        //
        // 获取到的appXXX=topic01, topic02, xxx, topicN; appXXY=
        // 首先通过;分割成列表，然后遍历找到appID
        apps := strings.Split(val, appsSeperator)
        for _, a := range apps {
            // 再给每个appid=topic01,topic02配置项进行解析
            // 通过'='分割，分割成appid与topic01, topic02...
            appTopics := strings.Split(a, appSeperator)
            if len(appTopics) == 0 {
                continue
            }

            // 然后在匹配appID与遍历检索到的appID是否相同
            app := appTopics[0]
            if app != appID {
                continue
            }

            // 最后再把topic01, topic02..., topic0N进行','分割，并返回列表给topic
            topics = strings.Split(appTopics[1], topicSeperator)
            break
        }
    }
    return topics
}

// GetAllowedTopics 用来校验允许进行发布和订阅的topic范围列表。
func GetAllowedTopics(metadata map[string]string) []string {
    topics := []string{}

    if val, ok := metadata[AllowedTopics]; ok && val != "" {
        topics = strings.Split(val, topicSeperator)
    }
    return topics
}
```

GetScopedTopics与GetAllowedTopics的关系说明：

对于任意一个发布和订阅的实例名称pubsub.name。从两个功能角度考虑：
1. publish。如果一个业务POD，其appid为appid1, 发布消息到指定的一个topic或者多个topic，则首先检测该topic是否在AllowedTopics；然后再检测该topic是否在PublishingScopes，配置了appid1=topic1, topicXXX..., 如果配置了，则表示可以进行publish；
2. subscribe。同上，只是对AllowdTopics检查完后，再对SubscriptionScopes的appid和topic进行校验，如果配置了，则表示可以进行subscribe

也就是说，AllowedTopics表示第一次拦截，ScopedTopics表示第二次拦截。当pubsub.name、appid和topic列表都通过后，则可以做相应的发布和订阅消息操作

**疑问**:

```yaml
  type: pubsub.redis
  version: v1
  metadata:
  - name: publishingScopes
    value: ""
```

下面这三个appid如果填写publishingScopes, 怎么填写呢？

|      | topic1 | topic2 | topic3 |
|------|--------|--------|--------|
| app1 |        |        |        |
| app2 |        |        |        |
| app3 |        |        |        |

## signals

```golang
// Context 用于接收系统信号，包括Ctrl+C中断进程等，并做一些context的后处理，包括尚未完成的业务逻辑、关闭监听端口等
// 然后再一次监听Ctrl+C中断进程信息后，再通过log fatal exit退出进程。可以拿来复用
//
// 因为这个进程中断与context绑定起来了。使得进程退出时可以做一些后处理。使得进程可以正常退出，尽量减少对业务的影响
func Context() context.Context {
    ctx, cancel := context.WithCancel(context.Background())
    sigCh := make(chan os.Signal)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    go func() {
        sig := <-sigCh
        log.Infof(`Received signal "%s"; beginning shutdown`, sig)
        cancel()
        sig = <-sigCh
        log.Fatalf(
            `Received signal "%s" during shutdown; exiting immediately`,
            sig,
        )
    }()
    return ctx
}
```

通过这种方式，可以减少流量的损失，以及对业务的损害值得我们学习

## cors

DefaultAllowedOrigins = "*"

这个cors包目前只有这行代码，表示默认支持所有跨域


## 总结

在上面这几个dapr包，对我最有收获的就是signals包，它与context联合来处理服务推出时，留出时间对业务处理，包括：未处理完的业务逻辑、关闭监听端口不再对外提供连接。以尽量减少对业务的负面影响
