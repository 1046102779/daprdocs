apis、client、http、metrics、proto、components、diagnostics以及testing包还没有进行源码阅读

本文针对components讲解, 首先针对整个components的加载和启动流程，以middleware/http为例，做一个整体流程的闭环梳理，然后在针对components每个类别做一个注册和使用的讲解。重点在components的加载、启动和使用流程讲解.


1. 针对components中的middleware/http类别组件，做一个整体的闭环梳理；
2. 在针对components包中的各个组件列表，进行注册源码阅读。

## middleware/http闭环

dapr为了使业务方可以可插拔使用任意组件类型和实例，采用了初始化全部一次性加载，然后业务方按需使用原则，进行创建。那么初始化加载过程， 即为组件**注册**过程；业务方通过yaml配置组件，则称为组件**创建**过程；

### 组件注册

组件注册，即为dapr runtime初始化阶段，一次性注册所有组件类型和实例到内存中。这个过程是通过`cmd/daprd/main.go`和`contrib-components`两部分相结合完成的, 在contrib-components中的middleware目前只有http，该middleware也是在fasthttp中装载拦截器链表使用。

下面先介绍contrib-components的middleware注册过程：

```golang
// cmd/daprd/main.go: main函数中加载所有components， 其中就有middleware的组件加载过程
//
// 下面uppercase, oauth2, oauth2clientcredentials, ratelimit, bearer和opa
// 这六种middleware/http拦截器，用于在处理dapr runtime接收外部http请求时，对请求数据进行拦截处理
//
// 这里注意两个方法：WithHTTPMiddleware和New
runtime.WithHTTPMiddleware(
  http_middleware_loader.New("uppercase", func(metadata middleware.Metadata) http_middleware.Middleware {
    return func(h fasthttp.RequestHandler) fasthttp.RequestHandler {
      return func(ctx *fasthttp.RequestCtx) {
        body := string(ctx.PostBody())
        ctx.Request.SetBody([]byte(strings.ToUpper(body)))
        h(ctx)
      }
    }
  }),
  http_middleware_loader.New("oauth2", func(metadata middleware.Metadata) http_middleware.Middleware {
    handler, _ := oauth2.NewOAuth2Middleware().GetHandler(metadata)
    return handler
  }),
  http_middleware_loader.New("oauth2clientcredentials", func(metadata middleware.Metadata) http_middleware.Middleware {
    handler, _ := oauth2clientcredentials.NewOAuth2ClientCredentialsMiddleware(log).GetHandler(metadata)
    return handler
  }),
  http_middleware_loader.New("ratelimit", func(metadata middleware.Metadata) http_middleware.Middleware {
    handler, _ := ratelimit.NewRateLimitMiddleware(log).GetHandler(metadata)
    return handler
  }),
  http_middleware_loader.New("bearer", func(metadata middleware.Metadata) http_middleware.Middleware {
    handler, _ := bearer.NewBearerMiddleware(log).GetHandler(metadata)
    return handler
  }),
  http_middleware_loader.New("opa", func(metadata middleware.Metadata) http_middleware.Middleware {
    handler, _ := opa.NewMiddleware(log).GetHandler(metadata)
    return handler
  }),
),
```

上面http_middleware_loader.New方法用于创建一个Middleware对象={组件列表实例名称Name={uppercase, ...}, FactorMethod}, 然后再通过WithHTTPMiddleware函数式传参方式，传入到dapr runtime对象中。就如下所示：

```golang
// WithHTTPMiddleware adds HTTP middleware components to the runtime.
func WithHTTPMiddleware(httpMiddleware ...http.Middleware) Option {
    return func(o *runtimeOpts) {
        o.httpMiddleware = append(o.httpMiddleware, httpMiddleware...)
    }
}
```

再把各个类别组件，作为多参数传入到DaprRuntime的Run方法中，所以最终这里有两层，一层为组件类别列表；另一层为具体组件类别下的组件实例列表，那么整个初始化源码，如下所示：

```golang
func (a *DaprRuntime) Run(opts ...Option) error {
    ......
    var o runtimeOpts
    // 根据传多参，进行组件类别列表的初始化，如： state、binding、middleware、pubsub......
    for _, opt := range opts {
        opt(&o)
    }

    // 在进行具体组件类别的实例列表注册过程, 如：middleware/http下的uppercase, oauth2, opa, ratelimit......
    err := a.initRuntime(&o)
    if err != nil {
        return err
    }
    ......
}

func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
  ......
  a.pubSubRegistry.Register(opts.pubsubs...)
  a.secretStoresRegistry.Register(opts.secretStores...)
  a.stateStoreRegistry.Register(opts.states...)
  a.bindingsRegistry.RegisterInputBindings(opts.inputBindings...)
  a.bindingsRegistry.RegisterOutputBindings(opts.outputBindings...)
  // 这里就是针对middleware/http的各个实例进行初始化注册过程
  //
  // 这里的opts.httpMiddleware...，就是前面的使用http_middleware_loader.New方法初始化的组件实例列表
  // 然后再把所有初始化的组件实例列表注册到dapr runtime内存中
  a.httpMiddlewareRegistry.Register(opts.httpMiddleware...)
  ......
  // 业务方加载yaml的创建和监听配置过程
  go a.processComponents()
  ......
  // 业务方对middleware/http各个组件实例的创建和监听配置过程
  pipeline, err := a.buildHTTPPipeline()
  ......
}
```

针对`a.httpMiddlewareRegistry.Register`各个实例注册过程，就是components/middeware/http的组件实例列表注册过程

```golang
type (
	// Middleware 要注册的middleware/http组件实例类型，包括组件实例名称, 如oauth2， uppercase，ratelimit等
  // 和初始化组件实例所需要的配置参数
  //
  // 这里重点讲解下FactoryMethod作用：
  // 初始化http_middleware.Middleware实例，可能需要配置参数，拿oauth2组件实例来说：
  // 当http header中的state参数值与目标state不匹配时，表示这个请求非法，可以重定向到指定的URL，或者
  // 通过http传入的code，和目标state，token_url, 或者auth_url, 生成一个新的token
  // 那么这些重定向其他服务，或者参数校验都需要借助于业务方在使用某个组件实例时，通过component yaml文件指定.spec.metadata来创建component实例对象
  // 而业务方使用过程，传入的yaml组件对象如下所示：
  /*
      apiVersion: dapr.io/v1alpha1
      kind: Component
      metadata:
        name: oauth2
      spec:
        ignoreErrors: true
        version: v1
        type: middleware.http.oauth2
        metadata:
        - name: clientID
          value: "xxx"
        - name: clientSecret
          value: "xxx"
        - name: scopes
          value: "*"
        - name: authURL
          value: "xxx"
        - name: tokenURL
          value: "xxx"
        - name: redirectURL
          value: "xxx"
  */
  // 那么上面.spec.metadata就是作为middleware/http中oauth2实例对象初始化的配置参数
  // 但并不是所有的组件类别，或者类别下的各个组件都需要配置参数。所以有些组件类别的FactoryMethod的初始化组件传入参数为空
  // 如下：FactoryMethod func() bindings.InputBinding
	Middleware struct {
		Name          string
		FactoryMethod func(metadata middleware.Metadata) http_middleware.Middleware
	}

	// Registry 作为组件实例初始化注册和业务方使用组件实例的创建使用接口
	Registry interface {
		Register(components ...Middleware)
		Create(name, version string, metadata middleware.Metadata) (http_middleware.Middleware, error)
	}

  // httpMiddlewareRegistry 作为Registry实现类，用来存储middleware/http各个组件
	httpMiddlewareRegistry struct {
		middleware map[string]func(middleware.Metadata) http_middleware.Middleware
	}
)

// New 创建一个组件实例
func New(name string, factoryMethod func(metadata middleware.Metadata) http_middleware.Middleware) Middleware {
	return Middleware{
		Name:          name,
		FactoryMethod: factoryMethod,
	}
}

// NewRegistry 返回一个middle/http实现类实例对象
func NewRegistry() Registry {
	return &httpMiddlewareRegistry{
		middleware: map[string]func(middleware.Metadata) http_middleware.Middleware{},
	}
}

// Register 把daprd main初始化加载所有的middeware/http组件实例列表，并放入到daprRuntime的initRuntime进行注册，并最终调用Register完成oauth2, uppercase, ratelimit, opa等组件实例的注册过程
func (p *httpMiddlewareRegistry) Register(components ...Middleware) {
  // 遍历middleware列表，注册到httpMiddlewareRegistry的middleware map内存中
	for _, component := range components {
		p.middleware[createFullName(component.Name)] = component.FactoryMethod
	}
}

// createFullName 组件实例唯一名称，它通过与路径保持一致，来达到唯一性
// 比如：
// middeware.http.upper，这个作为类型名称，而uppercase作为实例对象名
// 在业务方通过component或者configuration CRDs全局配置中的.spec.httpPipeline列表来创建实例并使用该组件实例
func createFullName(name string) string {
    return strings.ToLower("middleware.http." + name)
}
```

上面这一节以component-contrib中的middleware为例，介绍了dapr runtime对所有middleware/http组件实例列表的初始化和注册过程，接下来，我们再看业务方动态使用和加载middleware/http实例的使用过程。

### 组件创建

首先，先回顾下前面文章说到的dapr-system命名空间下dapr-operator服务提供的功能。它用于接收dapr CRDs创建dapr service，以及处理CRDs的自定义参数特性，并部署serivice和deployment。同时也对dapr runtime提供grpc服务，获取component列表、获取configuration配置、获取subscriptions、以及当component配置更新时主动通过grpc stream流推送给dapr runtime进行即使的更新和动态加载过程。

那么所有组件的创建过程则和dapr-operator强相关， 它通过拉取以及监听所有CRDs的动态变化，来获取yaml配置实现业务方的组件动态加载和删除、修改能力。搞清楚这点后，我们再看组件的整体创建过程

前面我们在daprRuntime的initRuntime方法中看到了下面两行代码，至于这两行的区别是什么，后面再说。目前只需要知道这两个都是针对业务需要的组件，进行创建。这里先讲解buildHTTPPipeline方法，它是对middlewear/http各个组件实例的创建过程， 而上文configuration文章已经对buildHTTPPipeline方法进行了分析

```golang
// 业务方加载yaml的创建和监听配置过程
go a.processComponents()
......
// 业务方对middleware/http各个组件实例的创建和监听配置过程
pipeline, err := a.buildHTTPPipeline()
......
```

我们只需要关注buildHTTPPipeline中的两行代码：

```golang
// 如果configuration yaml配置文件中自定义了一种dapr runtime不支持的组件类型或者具体的组件实例，则直接报错，说该组件暂不支持
component := a.getComponent(middlewareSpec.Type, middlewareSpec.Name)
......
// 然后再使用前面httpMiddlewareRegistry初始化好的Registry实例对象，进行创建。该Registry实例中已经注册了所有middlwware/http支持的组件实例
handler, err := a.httpMiddlewareRegistry.Create(middlewareSpec.Type, middle  wareSpec.Version,
                  middleware.Metadata{Properties: a.convertMetadataItemsToProperties(component.Spec.Metadata)})
```

接下来，就针对业务方要加载middleware/http各个实例的创建过程：

```golang
// Create 查找在httpMiddlewareRegistry中是否存在name和对应的version，如果存在，则把从dapr-system命名空间下通过dapr-operator服务拿到的component yaml配置下的.spec.metdata给到该实例组件,并初始化该实例
//
// 这样处理后，middleware/http中的各个组件实例，就可以通过pkg/http创建http服务时通过拦截器链表来对外提供http服务
func (p *httpMiddlewareRegistry) Create(name, version string, metadata middleware.Metadata) (http_middleware.Middleware, error) {
  // 注意，创建服务过程中，是通过middle.http.uppercase/version作为key
	if method, ok := p.getMiddleware(name, version); ok {
		return method(metadata), nil
	}
	return nil, errors.Errorf("HTTP middleware %s/%s has not been registered", name, version)
}

// getMiddleware 就是查找组件实例是否存在，这个传入的name就是configuration配置文件中的type，它是完整的路径，唯一性
// middleware.http.uppercase
func (p *httpMiddlewareRegistry) getMiddleware(name, version string) (func(middleware.Metadata) http_middleware.Middleware, bool) {
	nameLower := strings.ToLower(name)
	versionLower := strings.ToLower(version)
  // 这里比较有意思的一个点：key与版本号有关系
  // dapr 认为版本号为空，v0或者v1的版本，可以默认为空。表示不参与版本控制
  // 当版本号大于等于v2时，则需要强制带上版本号，则有具体实现就有了版本概念
	middlewareFn, ok := p.middleware[nameLower+"/"+versionLower]
	if ok {
		return middlewareFn, true
	}
	if components.IsInitialVersion(versionLower) {
		middlewareFn, ok = p.middleware[nameLower]
	}
	return middlewareFn, ok
}
```

这样，我们就完成了业务方加载和动态使用configuration具体实例过程。

## 其他components的注册过程

接下来，对其他component其他类别进行源码介绍和分析。通过上面middleware/http的注册和创建介绍，就比较容易理解其他components类型的注册和创建流程

### bindings

下面是针对InputBinding和OutputBinding，输入资源绑定和输出资源绑定，分别就对应着InputBinding的组件实例化和Outputbinding组件的实例化注册和创建过程，流程和middleware是完全一样，只是Registry接口多了一个组件实例是否存在的方法:`HasXXXBinding`

```golang
type (
      // InputBinding is an input binding component definition.
      InputBinding struct {
          Name          string
          FactoryMethod func() bindings.InputBinding
      }

      // OutputBinding is an output binding component definition.
      OutputBinding struct {
          Name          string
          FactoryMethod func() bindings.OutputBinding
      }

      // Registry is the interface of a components that allows callers to get registered   instances of input and output bindings
      Registry interface {
          RegisterInputBindings(components ...InputBinding)
          RegisterOutputBindings(components ...OutputBinding)
          HasInputBinding(name, version string) bool
          HasOutputBinding(name, version string) bool
          CreateInputBinding(name, version string) (bindings.InputBinding, error)
          CreateOutputBinding(name, version string) (bindings.OutputBinding, error)
      }

      bindingsRegistry struct {
          inputBindings  map[string]func() bindings.InputBinding
          outputBindings map[string]func() bindings.OutputBinding
      }
  )
```

### nameresolution

与middleware组件的注册和创建完全一样，只是nameresolution用于名字服务，用来解析appid||namespace，获取被调服务地址

### pubsub

同上

### secretstore

同上

### state

同上

但是state数据存储，需要使用到key，那么如何保证key的唯一性，这里就使用了state config来实现。接下来就是对state key的存储。目前key的存储，有四种方式：

1. 传入的key；
2. storeName||key
3. appId||key
4. 自定义的key前缀

至于采用那种方式作为state的key，则取决于component配置中的state.XXX 中的.spec.metdata中的keyPrefix值来决定的，keyPrefix值如下所示：
1. appid， 则state key最终存储的key为：appid||key;
2. none, 则state key最终存储的key为： key；
3. name， 则state key最终存储的key为：name||key;
4. 自定义值，则state key最终存储的key为： keyPrefix值||key.

yaml文件如下所示：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore01
spec:
  type: state.redis
  version: v1
  metadata:
  - keyPrefix: name
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
```

接下来的源码分析就是针对上面的流程：

```golang
const (
  // 也就是components中的state类型 keyPrefix存储的state key前缀
	strategyKey = "keyPrefix"

  // state key存储前缀的三种值，用来表示key的唯一性
	strategyAppid     = "appid"
	strategyStoreName = "name"
	strategyNone      = "none"
  // 如果配置文件中没有keyPrefix，则使用appid作为state key的前缀
	strategyDefault   = strategyAppid

	daprSeparator = "||"
)

// 存储dapr集群中所有业务components state实例对象
//
// map key作为.metdata.name, 也就是组件类型名称，而不是具体组件名称。
// 所以这里的key，是针对state组件类型名称的。只有一种statestore
//
// 而不能针对每个state具体实例单独设置key
var statesConfiguration = map[string]*StoreConfiguration{}

type StoreConfiguration struct {
	keyPrefixStrategy string
}

// SaveStateConfiguration 解析components中的.spec.metdata 前缀key值，并写入到statesConfiguration内存中
func SaveStateConfiguration(storeName string, metadata map[string]string) {
  // 如果component的statestore配置中，不带keyPrefix值, 则key使用appid值作为key的前缀
	strategy := metadata[strategyKey]
	strategy = strings.ToLower(strategy)
	if strategy == "" {
		strategy = strategyDefault
	}

  // 如果.spec.metadata中存在keyPrefix，则表示使用指定的key作为前缀，可以自定义key；
	statesConfiguration[storeName] = &StoreConfiguration{keyPrefixStrategy: strategy}
}

// GetModifiedStateKey 通过传入参数，来构建state存储key
func GetModifiedStateKey(key, storeName, appID string) string {
  // 通过yaml配置实例名称，来获取stateConfiguration
	stateConfiguration := getStateConfiguration(storeName)
  // 如果是strategyNone，则使用原生的key
  // 如果是strategyStoreName，则使用配置实例名称${storename}||key， 作为key
  // 如果是strategyAppid, 则使用业务APPID${appid}||key, 作为最终state存储的key
  // 如果上面三者都不是，则使用用户自定义的key作为state存储的最终key
	switch stateConfiguration.keyPrefixStrategy {
	case strategyNone:
		return key
	case strategyStoreName:
		return fmt.Sprintf("%s%s%s", storeName, daprSeparator, key)
	case strategyAppid:
		if appID == "" {
			return key
		}
		return fmt.Sprintf("%s%s%s", appID, daprSeparator, key)
	default:
    // 用户自定义的key前缀
		return fmt.Sprintf("%s%s%s", stateConfiguration.keyPrefixStrategy, daprSeparator, key)
	}
}

// GetOriginalStateKey 通过传入的state key，解析出用户传入的最初key
// 当前存储的key，有keyPrefix和用户传入的key，通过||字符串连接而成，那么索引为1的就是用户传入的原始key
func GetOriginalStateKey(modifiedStateKey string) string {
	splits := strings.Split(modifiedStateKey, daprSeparator)
	if len(splits) <= 1 {
		return modifiedStateKey
	}
	return splits[1]
}

// 解析statestore的唯一实例名称
func getStateConfiguration(storeName string) *StoreConfiguration {
	c := statesConfiguration[storeName]
	if c == nil {
		c = &StoreConfiguration{keyPrefixStrategy: strategyDefault}
		statesConfiguration[storeName] = c
	}

	return c
}
```

## 总结

本文主要是讲解components各个类型组件以及其下的各个具体实现组件的注册和用户创建过程，然后结合middleware/http的注册和创建来详细说明其整个流程，方便用户更容易理解。最后在components/state部分分析来state key的形成过程，用户可以通过component组件中的.spec.metdata.keyPrefix来设置state key的前缀，比如，redis key前缀可以是appid(微服务appid)，name(component配置实例唯一名称)、不含前缀的用户原生key，或者用户自定义的前缀key。
