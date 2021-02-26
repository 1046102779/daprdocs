dapr runtime http server服务下篇，主要接着api提供的剩余路由做一个介绍，然后再整体讲解http server的构建过程

```golang
// onPublish 主要是通过pubsub实例，来发布topic数据。
func (a *api) onPublish(reqCtx *fasthttp.RequestCtx)
  ......
  thepubsub := a.pubsubAdapter.GetPubSub(pubsubName)
  ......
  // 获取traceID，并把传入参数构建成cloudevent实例对象
  //
  // CloudEvents 是以通用格式描述事件数据的规范，以提供跨服务、平台和系统的互操作性。
  span := diag_utils.SpanFromContext(reqCtx)
  corID := diag.SpanContextToW3CString(span.SpanContext())
  envelope, err := runtime_pubsub.NewCloudEvent(&runtime_pubsub.CloudEvent{
    ID:              a.id,
    Topic:           topic,
    DataContentType: contentType,
    Data:            body,
    TraceID:         corID,
    Pubsub:          pubsubName,
  })
  // 获取pubsub的features列表，并查找在metadata中是否存在ttlInSeconds字段
  // 如果存在，则校验pubsub的features中是否关注该ttl， 这个表示cloud event对象在整个链路中的生存时间
  features := thepubsub.Features()
	pubsub.ApplyMetadata(envelope, features, metadata)
	b, err := a.json.Marshal(envelope)
  ......
  // 发布封装好的cloudevent实例对象到publishrequest中。
  req := pubsub.PublishRequest{
		PubsubName: pubsubName,
		Topic:      topic,
		Data:       b,
		Metadata:   metadata,
	}
	err = a.pubsubAdapter.Publish(&req)
  ......
}

// GetStatusCodeFromMetadata 校验metdata元数据中是否存在http.status_code字段，如果存在则错误码值给主调方
// 目前这个方法还没有被使用过
func GetStatusCodeFromMetadata(metadata map[string]string) int {
    code := metadata[http.HTTPStatusCode]
    if code != "" {
        statusCode, err := strconv.Atoi(code)
        if err == nil {
            return statusCode
        }
    }
    return fasthttp.StatusOK
}

// onGetHealthz 路由健康检查，检查http server是否就绪
func (a *api) onGetHealthz(reqCtx *fasthttp.RequestCtx) {
  if !a.readyStatus {
     // 返回健康检查没有就绪
  }
  // 否则，健康检查返回成功
  respondEmpty(reqCtx)
}

// onPostStateTransaction
func (a *api) onPostStateTransaction(reqCtx *fasthttp.RequestCtx) {
  ......
  // 请求数据中的key，进行keyPrefix前缀补充，并重新构建新的批量操作请求列表
  //
  // 并进行state事务提交, 这里需要注意，并不是所有components-contrib state具体组件接口的实现，都实现来Multi
  // API，而是有个默认的空state对象实例，继承关系，如果state具体组件没有实现该方法，则
  // 这个操作就是默认空操作
  err := transactionalStore.Multi(&state.TransactionalStateRequest{
      Operations: operations,
      Metadata:   req.Metadata,
  })
  respondEmpty(reqCtx)
}

// SetAppChannel 设置本dapr runtime访问本microservice的方式，包括：http和grpc
func (a *api) SetAppChannel(appChannel channel.AppChannel) {
    a.appChannel = appChannel
}

// SetDirectMessaging  设置service invocation直接调用实例对象，也就是内部业务微服务rpc方式
func (a *api) SetDirectMessaging(directMessaging messaging.DirectMessaging) {
    a.directMessaging = directMessaging
}

// SetActorRuntime 设置操作actor的实例对象
func (a *api) SetActorRuntime(actor actors.Actors) {
    a.actor = actor
}
```

## http server初始化和启动过程

http server对象初始化，通过NewServer创建Server实例对象，并通过StartNonBlocking方法来启动http端口服务, 启动端口服务时，还需要一些启动的配置，包括：是否开启metric、tracing、profiling等，以及是否需要进行http pipeline拦截器，对请求进行拦截处理。同时提供端口地址， appid等
```golang
type server struct {
	config      ServerConfig
	tracingSpec config.TracingSpec
	metricSpec  config.MetricSpec
	pipeline    http_middleware.Pipeline
	api         API
}

// NewServer 创建一个http server对象实例
func NewServer(api API, config ServerConfig, tracingSpec config.TracingSpec, metricSpec config.MetricSpec, pipeline http_middleware.Pipeline) Server {
	return &server{
		api:         api,
		config:      config,
		tracingSpec: tracingSpec,
		metricSpec:  metricSpec,
		pipeline:    pipeline,
	}
}

// StartNonBlocking 启动非阻塞的http server服务
func (s *server) StartNonBlocking() {
  // 构建拦截器链表，从外到里，逐个执行请求包处理
  //  处理流程：tracing->metric->apiauthentication->cors->components->router handler
  // 其中, apiauthentication是对http header中token进行校验，如果与环境变量：DAPR_API_TOKEN值相同，则通过
  // 否则拒绝, 响应无效token 401错误码
	handler :=
		useAPIAuthentication(
      // 跨域校验
			s.useCors(
        // components执行，也就是httpPipeline，在configuration设置
				s.useComponents(
          // 路由列表，也就是上篇中说到的Endpoints, 这背后就是对应的各个onXXXX handler
					s.useRouter())))

  // metrics和tracing, 分别是由metricspec中的Enabled和tracingSpec.SamplingRate开关控制
	handler = s.useMetrics(handler)
	handler = s.useTracing(handler)

	customServer := &fasthttp.Server{
		Handler:            handler,
		MaxRequestBodySize: s.config.MaxRequestBodySize * 1024 * 1024,
	}

  // 启动http server
	go func() {
		log.Fatal(customServer.ListenAndServe(fmt.Sprintf(":%v", s.config.Port)))
	}()

  // 是否开启http server性能监控
	if s.config.EnableProfiling {
		go func() {
			log.Infof("starting profiling server on port %v", s.config.ProfilePort)
			log.Fatal(fasthttp.ListenAndServe(fmt.Sprintf(":%v", s.config.ProfilePort), pprofhandler.PprofHandler))
		}()
	}
}

// unescapeRequestParametersHandler 主要解决路由过程中一些命名问题
// 比如：/v1.0/publish/{topic:*}, 有可能topic为"order%21shop",带有转义符号，去转义变为"order/shop"。这样在写入requestCtx时，需要对其值进行去转义。同样还有一个"order%20shop"去转义为"order shop"
func (s *server) unescapeRequestParametersHandler(next fasthttp.RequestHandler) fasthttp.RequestHandler {
	return func(ctx *fasthttp.RequestCtx) {
		parseError := false
    // 去转义方法
		unescapeRequestParameters := func(parameter []byte, value interface{}) {
			switch value.(type) {
			case string:
				if !parseError {
					parameterValue := fmt.Sprintf("%v", value)
          // 去转义
					parameterUnescapedValue, err := url.QueryUnescape(parameterValue)
					if err == nil {
						ctx.SetUserValueBytes(parameter, parameterUnescapedValue)
					} else {
						parseError = true
						errorMessage := fmt.Sprintf("Failed to unescape request parameter %s with value %v. Error: %s", parameter, value, err.Error())
						log.Debug(errorMessage)
						ctx.Error(errorMessage, fasthttp.StatusBadRequest)
					}
				}
			}
		}
    // 对requestCtx中的所有key-value的value值进行去转义
		ctx.VisitUserValues(unescapeRequestParameters)

		if !parseError {
			next(ctx)
		}
	}
}

// getRouter， 对Enpoints列表的所有路由进行处理，校验路由是否与"/{.*}"匹配
// 如果符合，则把该unescapeRequestParametersHandler方法设置为request handler上游
func (s *server) getRouter(endpoints []Endpoint) *routing.Router {
	router := routing.New()
	parameterFinder, _ := regexp.Compile("/{.*}")
	for _, e := range endpoints {
		path := fmt.Sprintf("/%s/%s", e.Version, e.Route)
		for _, m := range e.Methods {
      // 如果该路由命中了正则表达式，则需要对http请求url做去转义处理
			pathIncludesParameters := parameterFinder.MatchString(path)
			if pathIncludesParameters {
				router.Handle(m, path, s.unescapeRequestParametersHandler(e.Handler))
			} else {
				router.Handle(m, path, e.Handler)
			}
		}
	}
	return router
}
```

通过上下篇，我们就可以完整地了解dapr runtime http server的服务创建、拦截器链表、路由以及路由对应的handler处理过程。最终通过dapr runtime http server服务处理并转发给本地或者远端的microservices、或者转发给components-contrib组件库中初始化的第三方软件服务client真正处理外部请求。
