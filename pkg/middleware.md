目前还剩下apis、grpc、channel、client、http、metrics、proto、components、diagnostics和middleware，以及testing包还没有进行源码阅读

本文结合configuration章节，讲解middleware包。middleware包主要用于configuration yaml全局配置的httpPipline的注入过程，当外部请求经过dapr runtime http服务时，则通过链表把请求串行给各个handler处理，最后再交给转发给本daprd runtime所在的业务服务microservice处理。

接下来就通过`github.com/dapr/dapr/pkg/http`和`github.com/dapr/dapr/cmd/daprd`，以及configuration配置文件中的httpPipeline来处理

## middleware中间件

middleware是指daprd runtime http接收外部请求时，通过拦截器链表处理请求。

```golang
// http Middleware中间件，链表结构函数类型
type Middleware func(h fasthttp.RequestHandler) fasthttp.RequestHandler

// HTTPPipeline http pipeline 拦截器列表
type Pipeline struct {
    Handlers []Middleware
}

// BuildHTTPPipeline 无用方法, 可去掉
// 因为configuration配置中的httpPipeline需要使用到daprd runtime运行时，无法独立完成
func BuildHTTPPipeline(spec config.PipelineSpec) (Pipeline, error) {
    return Pipeline{}, nil
}

// Apply 把Pipeline中的拦截器列表链接成链表，索引为0的拦截器在链表首位。传入的handler在链表尾部
func (p Pipeline) Apply(handler fasthttp.RequestHandler) fasthttp.RequestHandler {
    for i := len(p.Handlers) - 1; i >= 0; i-- {
        handler = p.Handlers[i](handler)
    }
    return handler
}
```

## daprd components middleware加载

> github.com/dapr/dapr/cmd/daprd

在daprd runtime初始化启动过程中，首先会加载所有的components组件，这样业务服务在yaml配置中通过dapr-system命名空间下的dapr-operator服务动态加载引入时，就不用daprd runtime重启。这里就有了components有关middleware的加载过程, 也就是http的拦截器加载过程

目前contrib-components第三方组件库支持的middleware类型，可以参考：[middleware](https://github.com/dapr/components-contrib/tree/master/middleware), 拦截器有[bearer](https://github.com/dapr/components-contrib/tree/master/middleware/http/bearer), [nethttpadaptor](https://github.com/dapr/components-contrib/tree/master/middleware/http/nethttpadaptor), [oauth2](https://github.com/dapr/components-contrib/tree/master/middleware/http/oauth2), [oauth2clientcredentials](https://github.com/dapr/components-contrib/tree/master/middleware/http/oauth2clientcredentials), [opa](https://github.com/dapr/components-contrib/tree/master/middleware/http/opa), [ratelimit](https://github.com/dapr/components-contrib/tree/master/middleware/http/ratelimit)， 这些拦截器在http请求的处理作用各个功能和作用，后续再聊

以上contrib-components第三方组件库并不是全部middleware，还有一些内置初始化的组件，如下所示：

```golang
// 初始化加载所有的middleware到内存中，使得业务服务加载任何middleware配置，都可以直接找到该具体组件实例
runtime.WithHTTPMiddleware(
      // 这里就直接内置了一种middleware拦截器——uppercase
      // Middleware拦截器函数类型一个实例就是把请求包体数据改为小写
      /*
      // createFullName 则表示uppercase拦截器的唯一实例名称，也可以用来表示uppercase的路径，用于configuration配置httpPipeline表示type
      func createFullName(name string) string {
        return strings.ToLower("middleware.http." + name)
      }
      */
      // 全部通过pkg/components/middleware中的各个组件类型，分别进行初始化和注册过程，使得component配置加载时，可以根据type找到
      // 各个middleware拦截器实例的RequestHanlder
      http_middleware_loader.New("uppercase", func(metadata middleware.Metadata)   http_middleware.Middleware {
          return func(h fasthttp.RequestHandler) fasthttp.RequestHandler {
              return func(ctx *fasthttp.RequestCtx) {
                  body := string(ctx.PostBody())
                  ctx.Request.SetBody([]byte(strings.ToUpper(body)))
                  h(ctx)
              }
          }
      }),
      // 然后分别通过pkg/components/middleware初始化加载oatuh2， oauth2clientcredentials， ratelimit
      // bearer，opa
      http_middleware_loader.New("oauth2", func(metadata middleware.Metadata) htt  p_middleware.Middleware {
          handler, _ := oauth2.NewOAuth2Middleware().GetHandler(metadata)
          return handler
      }),
      http_middleware_loader.New("oauth2clientcredentials", func(metadata middlew  are.Metadata) http_middleware.Middleware {
          handler, _ := oauth2clientcredentials.NewOAuth2ClientCredentialsMiddlew  are(log).GetHandler(metadata)
          return handler
      }),
      http_middleware_loader.New("ratelimit", func(metadata middleware.Metadata)   http_middleware.Middleware {
          handler, _ := ratelimit.NewRateLimitMiddleware(log).GetHandler(metadata  )
          return handler
      }),
      http_middleware_loader.New("bearer", func(metadata middleware.Metadata) htt  p_middleware.Middleware {
          handler, _ := bearer.NewBearerMiddleware(log).GetHandler(metadata)
          return handler
      }),
      http_middleware_loader.New("opa", func(metadata middleware.Metadata) http_m  iddleware.Middleware {
          handler, _ := opa.NewMiddleware(log).GetHandler(metadata)
          return handler
      }),
  ),
```

如configuration配置文件如下所示：
```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: pipeline
spec:
  tracing:
    samplingRate: "1"
  httpPipeline:
    handlers:
    - type: middleware.http.uppercase
      name: uppercase
```

上面这个configuration yaml配置文件通过pkg/config配置中的k8s或者standalone方式进行配置加载，然后存储到daprd runtime的globalConfig中，最终通过`pkg/runtime/runtime.go`文件进行configuration的middleware拦截器初始化。

注意只有在configuration配置文件中写明了加载httpPipeline，才会在业务服务中使用到该middleware拦截器，下面针对**middleware.http.uppercase**进行源码阅读

```golang
// initRuntime 初始化daprd runtime
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error{
  ......
  pipeline, err := a.buildHTTPPipeline()
  ......
}

// buildHTTPPipeline 通过遍历configuration配置中的httpPipeline列表，来初始化RequestHandler到pipeline的列表中
func (a *DaprRuntime) buildHTTPPipeline() (http_middleware.Pipeline, error) {
    var handlers []http_middleware.Middleware

    if a.globalConfig != nil {
        // 如果globalConfig不为空, 则遍历configuration配置对象中.spec.httpPipeline.handlers列表
        for i := 0; i < len(a.globalConfig.Spec.HTTPPipelineSpec.Handlers); i++ {
            // 配置中的每个handler都是由{type, name, version，selector[{field, value}]}构成
            middlewareSpec := a.globalConfig.Spec.HTTPPipelineSpec.Handlers[i]
            // 前面daprd main初始化加载全部的components的middleware拦截器时，则可以通过middle.http.upppercase
            // 找到component实例对象
            // 如果是oauth2，则需要加载CRDs component组件实例名称为oauth2的组件，用于与第三方软件服务建立连接
            component := a.getComponent(middlewareSpec.Type, middlewareSpec.Name)
            if component == nil {
                return http_middleware.Pipeline{}, errors.Errorf("couldn't find middlew  are component with name %s and type %s/%s",
                    middlewareSpec.Name,
                    middlewareSpec.Type,
                    middlewareSpec.Version)
            }
            // 通过前面daprd main初始化component注册的middle.http.uppercase找到RequestHandler，并返回handler
            handler, err := a.httpMiddlewareRegistry.Create(middlewareSpec.Type, middle  wareSpec.Version,
                middleware.Metadata{Properties: a.convertMetadataItemsToProperties(comp  onent.Spec.Metadata)})
            if err != nil {
                return http_middleware.Pipeline{}, err
            }
            log.Infof("enabled %s/%s http middleware", middlewareSpec.Type, middlewareS  pec.Version)
            // 最终存储到handlers列表中
            handlers = append(handlers, handler)
        }
    }
    // 把handlers存储pipeline，并返回，至此configuration中的middleware加载、注册以及查找过程就全部弄清楚了
    return http_middleware.Pipeline{Handlers: handlers}, nil
}
```

上面buildHTTPPipeline方法，获取了业务服务关注的httpPipeline的middleware实例列表，但是这里还没有形成链表结构， 接下来讲述链表的建立过程，以及链表RequestHandler的队头和队尾插入过程

**注意： 目前configuration配置文件中的httpPipeline中的selector数据还没有开始启用**

## http pipeline的链表建立

> github.com/dapr/dapr/pkg/http

下面讲述daprd runtime对外提供http服务的启动过程中，httpPipeline加载过程。前面在daprd初始化，以及pkg/runtime/runtime.go中讲述了middleware的加载，httpPipeline配置加载，最终形成pipeline中的handlers列表过程

接下来，我们具体看daprd runtime http服务的拦截器构建成链表过程

```golang
func (s *server) StartNonBlocking() {
    // handler嵌套handler，本身比较晦涩，
    // 简明就是外层套里层，外层先执行， 那么handler封装下来最后形成一个完整的链表如下所示：
    //
    // useTracing->useMetrics->useAPIAuthentication->useCors->useComponents->useRouter
    // 而useComponents则是对configuration指定的httpPipeline handlers列表组件构建成正在的链表
    //
    // 在components中的middleware列表，索引为0的在useCors后面开头，
    // 而索引最大的len(handlers)在useRouter前面
    // 最终就形成了一个对外提供服务的拦截器处理请求链表
    //
    // middleware.Metadata作为component通用参数，当作闭包在pkg/runtime/runtime.go加载业务服务配置的httpPipeline实例时，通过component oauth2、ratelimit等实现类来传入.spec.metadata数据, 赋值给handler的内部对象初始化，使得可以根据metadata传入的参数动态处理http请求
    handler :=
        useAPIAuthentication(
            s.useCors(
                s.useComponents(
                    s.useRouter())))

    handler = s.useMetrics(handler)
    handler = s.useTracing(handler)

    customServer := &fasthttp.Server{
        Handler:            handler,
        MaxRequestBodySize: s.config.MaxRequestBodySize * 1024 * 1024,
    }

    go func() {
        log.Fatal(customServer.ListenAndServe(fmt.Sprintf(":%v", s.config.Port)))
    }()
    ......
}
```

通过上面的分析，我们可以完整的看到middleware包整个链路的组件middleware/http加载、component业务配置的引入和加载，以及daprd runtime提供http服务对所有拦截器的建立过程。
