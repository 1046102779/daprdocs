apis、client、http、metrics、proto、diagnostics以及testing包还没有进行源码阅读

本文讲解dapr runtime http包， 该包有两个用途：1. 指定的dapr runtime接收http请求；2. dapr runtime通过nameresolution名字服务，来查找目标dapr runtime服务地址，并判断目标dapr runtime是在本地，还是远端，以及第三方软件服务，比如components-contrib下的pubsub/binding等等。

1. 接收外部http路由请求；
2. 根据传入的appid以及namespace，通过components-contrib下的nameresolution名字服务，来获取目标地址；
3. 根据第二步骤，如果传入目标的appid和namespace，与当前接收http请求的appid和namespace相同，则表示就是该业务POD所在的业务微服务处理请求，也就是本地处理http请求
4. 根据第二步骤，如果目标appid和namespace，和接收这个请求处理的dapr runtime所在appid和namespace不匹配，则需要借助与nameresolution名字服务获取目标dapr runtime所在地址，并通过grpc进行rpc调用。
5. 经过第三步骤或者第四步骤，那么最终都会外部http请求，都会通过本地的dapr runtime访问本地的业务微服务

接下来就针对http api构建，以及dapr runtime http server服务的启动, 最终再通过一个http请求贯穿整个请求、转发处理流程

## http api路由

```golang
// API 接口用于构建http路由，以及设置dapr runtime http server是否准备就绪；
// 同时设置与dapr runtime与其绑定的业务微服务通信方式：grpc或者http， 也就是pod内的容器之间的通信
// DirectMessaging用于转发http请求到目标dapr runtime被调端
// actor稍后再看
type API interface {
	APIEndpoints() []Endpoint
	MarkStatusAsReady()
	SetAppChannel(appChannel channel.AppChannel)
	SetDirectMessaging(directMessaging messaging.DirectMessaging)
	SetActorRuntime(actor actors.Actors)
}

// http api作为API接口的一个具体实现
type api struct {
  // endpoints 表示http server路由列表
	endpoints             []Endpoint
  // directmessaging 表示接收dapr runtime的主调方，直接service invocation rpc调用目标dapr runtime，并最终调用业务微服务
	directMessaging       messaging.DirectMessaging
  // appChannel 用于设置本业务pod中的dapr runtime访问其中的microservice， 方式http或者grpc
	appChannel            channel.AppChannel
  // components组件列表， 后面再看::TODO
	components            []components_v1alpha1.Component
  // stateStores state存储map
	stateStores           map[string]state.Store
  // secretStores secret存储map
	secretStores          map[string]secretstores.SecretStore
  // secretScope
	secretsConfiguration  map[string]config.SecretsScope
  /json api
	json                  jsoniter.API
  // actor Actors接口的实现
	actor                 actors.Actors
  // pubsub适配器
	pubsubAdapter         runtime_pubsub.Adapter
  // 绑定
	sendToOutputBindingFn func(name string, req *bindings.InvokeRequest) (*bindings.InvokeResponse, error)
  // appid
	id                    string
	extendedMetadata      sync.Map
	readyStatus           bool
	tracingSpec           config.TracingSpec
}

// 创建一个API的api实例对象，并根据传入的参数进行初始化
func NewAPI(
	appID string,
	appChannel channel.AppChannel,
	directMessaging messaging.DirectMessaging,
	components []components_v1alpha1.Component,
	stateStores map[string]state.Store,
	secretStores map[string]secretstores.SecretStore,
	secretsConfiguration map[string]config.SecretsScope,
	pubsubAdapter runtime_pubsub.Adapter,
	actor actors.Actors,
	sendToOutputBindingFn func(name string, req *bindings.InvokeRequest) (*bindings.InvokeResponse, error),
	tracingSpec config.TracingSpec) API {
  // 初始化api
	api := &api{
		appChannel:            appChannel,
		directMessaging:       directMessaging,
		stateStores:           stateStores,
		secretStores:          secretStores,
		secretsConfiguration:  secretsConfiguration,
		json:                  jsoniter.ConfigFastest,
		actor:                 actor,
		pubsubAdapter:         pubsubAdapter,
		sendToOutputBindingFn: sendToOutputBindingFn,
		id:                    appID,
		tracingSpec:           tracingSpec,
	}
	api.components = components
  // 构建http server路由列表, 这些类型路由列表构建成了所有对外提供的dapr runtime http server服务
  // state 存储服务路由列表
	api.endpoints = append(api.endpoints, api.constructStateEndpoints()...)
  // secret 路由列表
	api.endpoints = append(api.endpoints, api.constructSecretEndpoints()...)
  // pubsub 发布订阅路由列表
	api.endpoints = append(api.endpoints, api.constructPubSubEndpoints()...)
  // actor Actor路由列表
	api.endpoints = append(api.endpoints, api.constructActorEndpoints()...)
  // directMessaging service invocation服务调用路由列表
	api.endpoints = append(api.endpoints, api.constructDirectMessagingEndpoints()...)
  // metadata 获取元数据路由列表
	api.endpoints = append(api.endpoints, api.constructMetadataEndpoints()...)
  // bindings 把事件传输给outputbinding路由列表
	api.endpoints = append(api.endpoints, api.constructBindingsEndpoints()...)
  // healthz 健康检查路由列表
	api.endpoints = append(api.endpoints, api.constructHealthzEndpoints()...)
  //
  // 以上八个路由类型列表，构成成了整个dapr runtime http server对外提供的所有服务，接下来就是针对这些路由背后的handler request处理请求进行源码阅读和分析
	return api
}

// APIEndpoints 返回dapr runtime http server路由列表
func (a *api) APIEndpoints() []Endpoint {
	return a.endpoints
}

// MarkStatusAsReady 标记http server准备就绪，可以对外提供服务
func (a *api) MarkStatusAsReady() {
	a.readyStatus = true
}

// 当前dapr runtime提供的http server都是v1.0版本，所以当前所有的外部http请求路由都是以
//
// /v1.0/xxx开头
//
// constructStateEndpoints 构建state路由列表
// 主要是对请求数据的CRUD，包括：批量新增、和事务提交操作
unc (a *api) constructStateEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/state/{storeName}/{key} GET  handler：onGetState
      // 作用：
      // 根据storeName和key，获取存储在dapr runtime中的storename与state type的映射关系，并发生第三方服务请求(components-contrib)
			Methods: []string{fasthttp.MethodGet},
			Route:   "state/{storeName}/{key}",
			Version: apiVersionV1,
			Handler: a.onGetState,
		},
		{
      // /v1.0/state/{storeName}  POST,PUT onPostState
      // 作用
      // 根据storeName找到component配置实例，并通过.spec.metdata找到配置，然后通过访问第三方state服务，来存储
      // key和value。比如：redis的键值对存储
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "state/{storeName}",
			Version: apiVersionV1,
			Handler: a.onPostState,
		},
		{
      // /v1.0/state/{storeName}/{key} DELETE onDeleteState
      // 作用：
      // 根据storeName找到state实例，并与state第三方服务通信，并删除state key数据
			Methods: []string{fasthttp.MethodDelete},
			Route:   "state/{storeName}/{key}",
			Version: apiVersionV1,
			Handler: a.onDeleteState,
		},
		{
      // /v1.0/state/{storename}/bulk POST, PUT, onBulkGetState
      // 作用：
      // 根据storeName找到第三方state软件服务，并进行批量存储操作
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "state/{storeName}/bulk",
			Version: apiVersionV1,
			Handler: a.onBulkGetState,
		},
		{
      // /v1.0/state/{storeName}/trasaction POST,PUT onPostStateTransaction
      // 作用：
      // 根据storeName找到第三方软件服务实例，然后再针对事务的一致性选择，来存储数据
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "state/{storeName}/transaction",
			Version: apiVersionV1,
			Handler: a.onPostStateTransaction,
		},
	}
}

// constructSecretEndpoints secrets路由列表，对secret密钥进行操作.
// 相当于请求访问所需要的资源所需要的token或者用户名、密码等
//
// 它包括单个获取和批量获取secrets
//
// 其他说明：
// 我们在dapr configuration一文中，详细介绍了.spec.secrets.scopes={storeName, defaultAccess, allowedSecrets, deniedSecrets}
func (a *api) constructSecretEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/secrets/{secretStoreName}/bulk GET onBulkGetSecret
      // 作用
      // 通过secretStoreName找到配置实例的存储对象，然后获取这个存储对象下的secrets
      // 然后再经过configuration对象的.spec.secrets.scopes下的storeName进行允许或者拒绝的secrets action匹配
      // 匹配成功的secrets可以返回
      //
      // 最终这个返回对象就是包含了secret key，以及对应value map，因为一个key，信息可能会存在多个。
      // 比如：key=account(业务名)， value={username: seachen, passwd: 123456}
			Methods: []string{fasthttp.MethodGet},
			Route:   "secrets/{secretStoreName}/bulk",
			Version: apiVersionV1,
			Handler: a.onBulkGetSecret,
		},
		{
      // /v1.0/secrets/{secretStoreName}/{key} GET onGetSecret
      // 作用：
      // 通过secretStoreName名称找到配置实例的存储对象, 然后再获取这个存储对象下的指定key所对应的secret信息
			Methods: []string{fasthttp.MethodGet},
			Route:   "secrets/{secretStoreName}/{key}",
			Version: apiVersionV1,
			Handler: a.onGetSecret,
		},
	}
}

// constructPubSubEndpoints pubsub消息发布路由
func (a *api) constructPubSubEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/publish/{pubsubname}/{topic:*} POST,PUT onPublish
      // 作用：
      // 根据传入的pubsubname，找到components下的pubsub实例名称，并对topic进行消息发布
      // 而具体的onPublish中，针对pubsub适配器接口的实现，其实就是dapr runtime实现的Publish和GetPubsub
      //
      // 注意这里一个比较有意思的点：{topic:*}
      // 它是针对topic值的转义。目前支持：空格和反斜杠： ' '和'/'
      // 因为topic命名可能是={unknown%20state%20store, stateStore%2F1}
      // 转义出来的结果={unknown state store, stateStore/1}
      // 具体是如何实现的，后面再看
      // 注意pubsub消息传递使用的是标准的cloudEvent
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "publish/{pubsubname}/{topic:*}",
			Version: apiVersionV1,
			Handler: a.onPublish,
		},
	}
}

// constructBindingsEndpoints 构建output binding路由
func (a *api) constructBindingsEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/bindings/{name} POST, PUT onOutputBindingMessage
      // 作用：
      // 使用name找到output binding对应的实例，然后把事件数据推送给第三方软件服务
      // 针对outputbinding和inputbinding的说明：
      // 1. binding是外部资源；
      // 2. outputbinding：允许microservice通过outputbinding访问外部资源，也就是bindings
      // 3. inputbinding：允许microservice通过inputbinding接收外部资源。
      // 所以input和output都是针对microservice的。是接收绑定的外部资源，还是把资源绑定发布给外部资源
      //
      // 所以这里的outputbinding是指microservice调用外部资源
      // inputbinding，允许inputbinding接收外部资源
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "bindings/{name}",
			Version: apiVersionV1,
			Handler: a.onOutputBindingMessage,
		},
	}
}

// constructDirectMessagingEndpoints 是direct messaging(service invocation) microservice rpc直接调用
func (a *api) constructDirectMessagingEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/invoke/{id}/method/{method:*}  允许所有的方法*  onDirectMessage
      // 作用：
      // 根据id和nameresolution找到目标dapr runtime地址，然后再判断目标服务是在本地，还是在远端
      // {method:*} 与前面的{topic:*}是一样的处理方式，表示method方法可以是任意字符串，包括'/'和' '
			Methods: []string{router.MethodWild},
			Route:   "invoke/{id}/method/{method:*}",
			Version: apiVersionV1,
			Handler: a.onDirectMessage,
		},
	}
}

// constructActorEndpoints 用于构建actors的访问和在线实时任务调度处理
func (a *api) constructActorEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/actors/{actorType}/{actorId}/state POST,PUT onActorStateTransaction
      // 作用：
      // 根据actorType和actorId，找到dapr runtime下actorId的实例，并把批量的状态数据存储到该state中
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "actors/{actorType}/{actorId}/state",
			Version: apiVersionV1,
			Handler: a.onActorStateTransaction,
		},
		{
      // /v1.0/actors/{actorType}/{actorId}/method/{method} * onDirectActorMessage
      // 作用：
      // 根据actorType和actorId找到dapr runtime的actorId实例，并转发给microservice的method处理
      // DirectActorFMessage和DirectMessage两个不同之处，在于actorId获取的请求是串行执行，而rpc直接调用可以支持并发
			Methods: []string{fasthttp.MethodGet, fasthttp.MethodPost, fasthttp.MethodDelete, fasthttp.MethodPut},
			Route:   "actors/{actorType}/{actorId}/method/{method}",
			Version: apiVersionV1,
			Handler: a.onDirectActorMessage,
		},
		{
      // /v1.0/actors/{actorType}/{actorId}/state/{key} GET onGetActorState
      // 作用
      // 根据actorType和actorId找到dapr runtime的actorId，并获取actor存储的key对应的值
			Methods: []string{fasthttp.MethodGet},
			Route:   "actors/{actorType}/{actorId}/state/{key}",
			Version: apiVersionV1,
			Handler: a.onGetActorState,
		},
		{
      // /v1.0/actors/{actorType}/{actorId}/reminders/{name} POST, PUT, onCreateActorReminder
      // 作用：
      // 根据actorType和actorId，找到dapr runtime下的actorId对象, 然后根据reminder name创建数据
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "actors/{actorType}/{actorId}/reminders/{name}",
			Version: apiVersionV1,
			Handler: a.onCreateActorReminder,
		},
		{
      // /v1.0/actors/{actorType}/{actorId}/timers/{name} POST,PUT onCreateActorTimer
      // 作用：
      // 根据actorType和actorId，找到dapr runtime下的actorId对象，然后再提交timer name创建数据
      //
      // timer和reminder不同点，在于前者随着actor的销毁而销毁；而后者是持久化存储的，不再执行时，需要用户主动删除timer
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "actors/{actorType}/{actorId}/timers/{name}",
			Version: apiVersionV1,
			Handler: a.onCreateActorTimer,
		},
		{
      // 删除reminder动作
			Methods: []string{fasthttp.MethodDelete},
			Route:   "actors/{actorType}/{actorId}/reminders/{name}",
			Version: apiVersionV1,
			Handler: a.onDeleteActorReminder,
		},
		{
      // 删除timer动作
			Methods: []string{fasthttp.MethodDelete},
			Route:   "actors/{actorType}/{actorId}/timers/{name}",
			Version: apiVersionV1,
			Handler: a.onDeleteActorTimer,
		},
		{
      // timers是不需要持久化的，业务pod销毁则timer销毁；而reminder则是需要持久化，需要读取的和重新分配的
      //
      // 这里详细说下为什么有onGetActorReminder，而没有onGetActorTimer
      // 即
      // /v1.0/actors/{actorType}/{actorId}/reminders/{name}存在的理由
      // 和
      // /v1.0/actors/{actorType}/{actorId}/timers/{name}不存在的理由
      //
      // 如果microservice01 运行reminder01， 如果该服务销毁，则reminder01会被dapr-system命名空间下的dapr-placement调度到其他microservice中，再接着执行。所以外界资源需要GetReminder API的，来获取reminder来继承原来的actor。actorType和actorId不变；
      // 而microservice01 运行timer01，它会随着服务的销毁而销毁，再也不会被调度，所以一旦运行，接下来的操作只有删除销毁掉。不需要外界干预和继承
      // 这个站不站得住脚，后面再看::TODO
			Methods: []string{fasthttp.MethodGet},
			Route:   "actors/{actorType}/{actorId}/reminders/{name}",
			Version: apiVersionV1,
			Handler: a.onGetActorReminder,
		},
	}
}

// constructMetadataEndpoints 用于获取dapr runtime http内部的metdata元数据相关信息
// 包括：appid, 当前dapr runtime正在运行的actor梳理、已经注册的components列表，以及扩展元数据
func (a *api) constructMetadataEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/metdata GET
      //返回dapr runtime http server所有的元数据
			Methods: []string{fasthttp.MethodGet},
			Route:   "metadata",
			Version: apiVersionV1,
			Handler: a.onGetMetadata,
		},
		{
      // /v1.0/metadata/{key} PUT
      // 存储外部传入的键值对数据。一般是存储同一个pod中的microservice数据，因为dapr runtime也是业务微服务的一个运行时，减轻业务负担。存储到扩展元数据中
			Methods: []string{fasthttp.MethodPut},
			Route:   "metadata/{key}",
			Version: apiVersionV1,
			Handler: a.onPutMetadata,
		},
	}
}

// constructHealthzEndpoints 健康检查路由列表服务
func (a *api) constructHealthzEndpoints() []Endpoint {
	return []Endpoint{
		{
      // /v1.0/healthz GET onGetHealthz
      // 作用
      // dapr runtime http server的健康检查。
			Methods: []string{fasthttp.MethodGet},
			Route:   "healthz",
			Version: apiVersionV1,
			Handler: a.onGetHealthz,
		},
	}
}
```

## http api handler

针对http api路由列表对应的各个request handler处理函数。

### outputbinding handler

```golang
// onOutputBindingMessage microservice调用外部资源, 这个主要业务逻辑都在daprRuntime的sendTooutputbinding方法中
//
// 其他操作都是针对error以及获取请求数据、序列化和反序列化，以及获取trace等
func (a *api) onOutputBindingMessage(reqCtx *fasthttp.RequestCtx) {
  // 路由：/v1.0/bindings/{name}
  // 比如：curl -X POST -H  http://localhost:3500/v1.0/bindings/myevent -d '{ "data": { "message": "Hi!" }, "operation": "create" }'
  // 获取restful风格路由的name值以及获取request body数据
	name := reqCtx.UserValue(nameParam).(string)
	body := reqCtx.PostBody()

  // 把request body反序列化为OutputBindingRequest实例对象
	var req OutputBindingRequest
	err := a.json.Unmarshal(body, &req)
	if err != nil {
		msg := NewErrorResponse("ERR_MALFORMED_REQUEST", fmt.Sprintf(messages.ErrMalformedRequest, err))
		respondWithError(reqCtx, fasthttp.StatusBadRequest, msg)
		log.Debug(msg)
		return
	}

	b, err := a.json.Marshal(req.Data)
	if err != nil {
		msg := NewErrorResponse("ERR_MALFORMED_REQUEST_DATA", fmt.Sprintf(messages.ErrMalformedRequestData, err))
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		log.Debug(msg)
		return
	}

	// 从requestcontext上下文尝试获取trace span，如果能获取到则需要通过outputbinding的metdata元数据
  // 传递给binding的外部资源,  元数据key：traceparent， value为:trace相关信息
	if span := diag_utils.SpanFromContext(reqCtx); span != nil {
		sc := span.SpanContext()
		if req.Metadata == nil {
			req.Metadata = map[string]string{}
		}
		req.Metadata[traceparentHeader] = diag.SpanContextToW3CString(sc)
		if sc.Tracestate != nil {
			req.Metadata[tracestateHeader] = diag.TraceStateToW3CString(sc)
		}
	}

  // sendToOutputBindingFn 为daprRuntime中的sendToOutputBinding方法
  // 并发起OutputBinding的Invoke操作
  //
  // 我们看到所有和第三方components-contrib服务的通信，都是通过daprRuntime处理的。
	resp, err := a.sendToOutputBindingFn(name, &bindings.InvokeRequest{
		Metadata:  req.Metadata,
		Data:      b,
		Operation: bindings.OperationKind(req.Operation),
	})
	if err != nil {
		msg := NewErrorResponse("ERR_INVOKE_OUTPUT_BINDING", fmt.Sprintf(messages.ErrInvokeOutputBinding, name, err))
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		log.Debug(msg)
		return
	}
	if resp == nil {
		respondEmpty(reqCtx)
	} else {
		respondWithJSON(reqCtx, fasthttp.StatusOK, resp.Data)
	}
}

// onBulkGetState 获取指定component下state storename传入的批量key，然后获取该存储下这些批量key下对应的所有数据列表
//
// 需要注意的一个问题是，可能state类别的所有具体存储组件，有些并不支持批量操作，这个就需要一个个上报
// 不支持批量操作，如果单独一个个上报，则耗时太长，这里就需要前面文章提到的concurrency包
// 它用来控制goroutine的并发量，来平衡退而求次的批量操作。
func (a *api) onBulkGetState(reqCtx *fasthttp.RequestCtx) {
  //  /v1.0/state/{storeName}/bulk
  // curl -X POST -H "Content-Type: application/json" -d '{"keys":["key1", "key2"]}' http://localhost:3500/v1.0/state/statestore/bulk
  //
  // 从请求路由获取storeName和api存储的stateStores对应的第三方存储对象
	store, storeName, err := a.getStateStoreWithRequestValidation(reqCtx)
  // 如果stateStores长度为0，或者storeName没有发现，都报错
	if err != nil {
		log.Debug(err)
		return
	}

  // 请求数据反序列化为BulkGetRequest实例对象
  /*
    type BulkGetRequest struct {
      Metadata    map[string]string `json:"metadata"`
      Keys        []string          `json:"keys"`
      // 如果storeName映射的第三方存储服务不支持批量操作，则需要设置单个操作的最大并发量, 使得可以快速响应请求
      Parallelism int               `json:"parallelism"`
    }
  */
	var req BulkGetRequest
	err = a.json.Unmarshal(reqCtx.PostBody(), &req)
	if err != nil {
		msg := NewErrorResponse("ERR_MALFORMED_REQUEST", fmt.Sprintf(messages.ErrMalformedRequest, err))
		respondWithError(reqCtx, fasthttp.StatusBadRequest, msg)
		log.Debug(msg)
		return
	}

  // 这里针对url的参数对参数key去'metdata.'
  // 如：metdata.name=seachen&value=tencent
  // 则处理后的结果为{name: seachen, value: tencent}
	metadata := getMetadataFromRequest(reqCtx)

  // 如果传入的keys列表为空，则直接返回响应空数据
	bulkResp := make([]BulkGetResponse, len(req.Keys))
	if len(req.Keys) == 0 {
		b, _ := a.json.Marshal(bulkResp)
		respondWithJSON(reqCtx, fasthttp.StatusOK, b)
		return
	}

	// 请求参数转换，在上文中聊过的components/state中state_config.go文件中，针对key的前缀修改
  // 目前支持的前缀，是通过components组件statestore中.spec.metadata列表，获取keyPrefix值
  // keyPrefix值，有none、appid、storename和自定义的key前缀
  // 而state_loader.GetModifiedStateKey方法就是来修改前缀的。
	reqs := make([]state.GetRequest, len(req.Keys))
	for i, k := range req.Keys {
		r := state.GetRequest{
			Key:      state_loader.GetModifiedStateKey(k, storeName, a.id),
			Metadata: req.Metadata,
		}
		reqs[i] = r
	}
  // 批量获取指定存储的key列表所对应的value列表
  // 返回的第一个参数bulkGet表示是否支持批量操作。
	bulkGet, responses, err := store.BulkGet(reqs)

  // 如果支持, 则对response数据列表进行数据校验
	if bulkGet {
		// 如果批量操作失败，则直接响应返回错误
		if err != nil {
			msg := NewErrorResponse("ERR_MALFORMED_REQUEST", fmt.Sprintf(messages.ErrMalformedRequest, err))
			respondWithError(reqCtx, fasthttp.StatusBadRequest, msg)
			log.Debug(msg)
			return
		}

    // 否则，遍历response列表，并获取最初用户传入的key
    // 并校验response的error是否为空，并赋值数据给bulkResp列表
		for i := 0; i < len(responses) && i < len(req.Keys); i++ {
			bulkResp[i].Key = state_loader.GetOriginalStateKey(responses[i].Key)
			if responses[i].Error != "" {
				log.Debugf("bulk get: error getting key %s: %s", bulkResp[i].Key, responses[i].Error)
				bulkResp[i].Error = responses[i].Error
			} else {
				bulkResp[i].Data = jsoniter.RawMessage(responses[i].Data)
				bulkResp[i].ETag = responses[i].ETag
			}
		}
	} else {
    // 如果该存储不支持BulkGet操作，则通过concurrency并发度控制goroutine并发量
    // 批量执行获取单个key的API操作
    //
    // 注意如果用户没有传入parallelism，则默认并发量为100
		limiter := concurrency.NewLimiter(req.Parallelism)

    // 获取单个key所存储的值，并进行response数据转换
		for i, k := range req.Keys {
			bulkResp[i].Key = k

      // 基本上可以批量操作相同
			fn := func(param interface{}) {
				r := param.(*BulkGetResponse)
				gr := &state.GetRequest{
					Key:      state_loader.GetModifiedStateKey(r.Key, storeName, a.id),
					Metadata: metadata,
				}

				resp, err := store.Get(gr)
				if err != nil {
					log.Debugf("bulk get: error getting key %s: %s", r.Key, err)
					r.Error = err.Error()
				} else if resp != nil {
					r.Data = jsoniter.RawMessage(resp.Data)
					r.ETag = resp.ETag
				}
			}

			limiter.Execute(fn, &bulkResp[i])
		}
		limiter.Wait()
	}

  // 最终把批量响应数据返回给主调方
	b, _ := a.json.Marshal(bulkResp)
	respondWithJSON(reqCtx, fasthttp.StatusOK, b)
}
// 以上需要注意的是，如果storeName对应的第三方存储服务支持事务提交和查询操作，则可以直接采用Multi方法进行操作；
// 如果不支持，则采用退而求次的BulkGet方法，如果第三方存储服务的BulkGet实现为空，则bulkGet返回一定是false，这样
// 最终再采用one by one的操作来退而求次的实现dapr runtime http server 的BulkGet方法。

// /v1.0/state/{storeName}/{key}
// onGetState 通过
func (a *api) onGetState(reqCtx *fasthttp.RequestCtx) {
  // 同上onBulkGetState， 获取路由上的storeName，以及对应的第三方存储实例
	store, storeName, err := a.getStateStoreWithRequestValidation(reqCtx)
	if err != nil {
		log.Debug(err)
		return
	}

  // 把url请求参数中的'metdata.'前缀去掉，并把参数key和value构建成map
	metadata := getMetadataFromRequest(reqCtx)

  // 获取路由上的{key}映射的值
	key := reqCtx.UserValue(stateKeyParam).(string)
  // 查找url请求参数中，是否含有consistency策略: strong和eventual两种策略
  // dapr runtime这里并没有对一致性策略值做校验，而是直接丢给第三方存储服务校验该consistency参数值
	consistency := string(reqCtx.QueryArgs().Peek(consistencyParam))
  // 创建访问第三方state存储服务的请求实例
	req := state.GetRequest{
		Key: state_loader.GetModifiedStateKey(key, storeName, a.id),
		Options: state.GetStateOption{
			Consistency: consistency,
		},
		Metadata: metadata,
	}

  // 和前面onBulkGetState的bulkGet为false时one by one的操作
	resp, err := store.Get(&req)
	if err != nil {
		storeName := a.getStateStoreName(reqCtx)
		msg := NewErrorResponse("ERR_STATE_GET", fmt.Sprintf(messages.ErrStateGet, key, storeName, err.Error()))
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		log.Debug(msg)
		return
	}
  // 返回响应数据给主调方
	if resp == nil || resp.Data == nil {
		respondEmpty(reqCtx)
		return
	}
	respondWithETaggedJSON(reqCtx, fasthttp.StatusOK, resp.Data, resp.ETag)
}

// /v1.0/secrets/{secretStoreName}/{key}
// onGetSecret 获取指定secretStoreName实例存储下的key(可以是业务名)对应的相关密钥信息，包括用户名、密码、token等等
func (a *api) onGetSecret(reqCtx *fasthttp.RequestCtx) {
	// 与onGetStateStore类似，获取secreteStoreName名称对应的secret存储实例
	store, secretStoreName, err := a.getSecretStoreWithRequestValidation(reqCtx)
	if err != nil {
		log.Debug(err)
		return
	}

	// 请求url参数key带有'metadata.'去掉
	// 如metdata.username为username，构建成元数据map存储
	metadata := getMetadataFromRequest(reqCtx)

	// 从restful url上获取secret存储实例的key
	key := reqCtx.UserValue(secretNameParam).(string)

	// 并校验该secretStoreName实例下的seret key是否允许访问
	// 比如key为daprsecret、redissecret等等
	//
	// 如果该secret实例下的key不允许被访问，则直接返回403错误码
	if !a.isSecretAllowed(secretStoreName, key) {
		msg := NewErrorResponse("ERR_PERMISSION_DENIED", fmt.Sprintf(messages.ErrPermissionDenied, key, secretStoreName))
		respondWithError(reqCtx, fasthttp.StatusForbidden, msg)
		return
	}

	// 通过key和metadata，获取secret实例下的指定key的密钥数据
	req := secretstores.GetSecretRequest{
		Name:     key,
		Metadata: metadata,
	}
	resp, err := store.GetSecret(req)
	if err != nil {
		msg := NewErrorResponse("ERR_SECRET_GET",
			fmt.Sprintf(messages.ErrSecretGet, req.Name, secretStoreName, err.Error()))
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		log.Debug(msg)
		return
	}

	// 返回响应数据
	if resp.Data == nil {
		respondEmpty(reqCtx)
		return
	}

	respBytes, _ := a.json.Marshal(resp.Data)
	respondWithJSON(reqCtx, fasthttp.StatusOK, respBytes)
}

// /v1.0/secrets/{secretStoreName}/bulk
// onBulkGetSecret 批量获取seretStoreName名称关联的secret存储实例
func (a *api) onBulkGetSecret(reqCtx *fasthttp.RequestCtx) {
	// 同上onGetSecret, 获取secretStoreName名称关联的secret实例， 如果该secretStoreName不存在，则返回错误
	store, secretStoreName, err := a.getSecretStoreWithRequestValidation(reqCtx)
	if err != nil {
		log.Debug(err)
		return
	}

	// 清除url携带的带有metdata.的前缀
	metadata := getMetadataFromRequest(reqCtx)

	// 批量获取secret, 比如获取secretStoreName实例下的所有密钥数据，如：daprsecret, redissecret, mysqlsecret等
	req := secretstores.BulkGetSecretRequest{
		Metadata: metadata,
	}

	// 如果获取失败，则返回500错误码
	resp, err := store.BulkGetSecret(req)
	if err != nil {
		msg := NewErrorResponse("ERR_SECRET_GET",
			fmt.Sprintf(messages.ErrBulkSecretGet, secretStoreName, err.Error()))
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		log.Debug(msg)
		return
	}

	if resp.Data == nil {
		respondEmpty(reqCtx)
		return
	}

	// 对返回的secret存储的密钥数据列表，遍历判断每个secret密钥数据，是否允许返回给上游
	// 这个是在configuration一文中有详细说明
	// 比如：
	// {redissecret: {"auth": "xxxx", "address": "xxx"}}等等
	filteredSecrets := map[string]map[string]string{}
	for key, v := range resp.Data {
		if a.isSecretAllowed(secretStoreName, key) {
			filteredSecrets[key] = v
		} else {
			// 如果key被deny，则落日志
			log.Debugf(messages.ErrPermissionDenied, key, secretStoreName)
		}
	}

	// 把secretStoreName名称对应的secret实例下的所有allow的secret密钥数据，返回给主调方
	respBytes, _ := a.json.Marshal(filteredSecrets)
	respondWithJSON(reqCtx, fasthttp.StatusOK, respBytes)
}

// /v1.0/state/{storeName}
// onPostState 把一个或者多个数据存储到storeName名称对应的实例中
func (a *api) onPostState(reqCtx *fasthttp.RequestCtx) {
	// 同上，获取storeName，以及校验当前dapr runtime http server是否支持storeName实例
	store, storeName, err := a.getStateStoreWithRequestValidation(reqCtx)
	if err != nil {
		log.Debug(err)
		return
	}

	// 解析state存储数据所需要的键值对，以及metdata和操作策略
	// 目前存储数据的选项，包括：数据一致性和并发策略。
	// 数据一致性，包括：最终一致性，和强一致性；
	// 并发：first-write, last-write
	// first-write: 并发情况下，只有第一个能写入成功，实现机制CAS
	// last-write: 无需考虑并发，请求来了就写入
	reqs := []state.SetRequest{}
	err = a.json.Unmarshal(reqCtx.PostBody(), &reqs)
	if err != nil {
		msg := NewErrorResponse("ERR_MALFORMED_REQUEST", err.Error())
		respondWithError(reqCtx, fasthttp.StatusBadRequest, msg)
		log.Debug(msg)
		return
	}

	// 针对每个key-value键值对存储，进行keyPrefix的修改
	for i, r := range reqs {
		reqs[i].Key, err = state_loader.GetModifiedStateKey(r.Key, storeName, a.id)
		if err != nil {
			msg := NewErrorResponse("ERR_MALFORMED_REQUEST", err.Error())
			respondWithError(reqCtx, fasthttp.StatusBadRequest, msg)
			log.Debug(err)
			return
		}
	}

	// 发起第三方服务调用，存储批量的kv键值对，并带有元数据和一致性、并发策略选项
	err = store.BulkSet(reqs)
	if err != nil {
		storeName := a.getStateStoreName(reqCtx)

		statusCode, errMsg, resp := a.stateErrorResponse(err, "ERR_STATE_SAVE")
		resp.Message = fmt.Sprintf(messages.ErrStateSave, storeName, errMsg)

		respondWithError(reqCtx, statusCode, resp)
		log.Debug(resp.Message)
		return
	}

	respondEmpty(reqCtx)
}

// /v1.0/invoke/{id}/method/{method:*}
// onDirectMessage 接收外部请求，包括来自其他dapr runtime或者microservice的请求，进行rpc直接调用
// 注意，这个内部的实现，我们在messaging一文中详细说明了。
func (a *api) onDirectMessage(reqCtx *fasthttp.RequestCtx) {
	// 获取路由中的id值和method值，这样就可以知道appid和appid对应的microservice中handler
	targetID := reqCtx.UserValue(idParam).(string)
	verb := strings.ToUpper(string(reqCtx.Method()))
	invokeMethodName := reqCtx.UserValue(methodParam).(string)

	// 如果该appid所在的microservice没有提供对外的服务，则直接报错
	if a.directMessaging == nil {
		msg := NewErrorResponse("ERR_DIRECT_INVOKE", messages.ErrDirectInvokeNotReady)
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		return
	}

	// 构建service invocation的请求参数实例
	req := invokev1.NewInvokeMethodRequest(invokeMethodName).WithHTTPExtension(verb, reqCtx.QueryArgs().String())
	req.WithRawData(reqCtx.Request.Body(), string(reqCtx.Request.Header.ContentType()))
	// Save headers to internal metadata
	req.WithFastHTTPHeaders(&reqCtx.Request.Header)

	// directmessaging通过Invoke内部的nameresolution名字服务，发现appid和namespace的目标地址
	// 再校验appid是否为本dapr runtime所在的microservice。如果是直接本地调用；
	// 否则发起本dapr runtime访问目标dapr runtime的grpc调用
	resp, err := a.directMessaging.Invoke(reqCtx, targetID, req)
	if err != nil {
		statusCode := fasthttp.StatusInternalServerError
		if status.Code(err) == codes.PermissionDenied {
			statusCode = invokev1.HTTPStatusFromCode(codes.PermissionDenied)
		}
		msg := NewErrorResponse("ERR_DIRECT_INVOKE", fmt.Sprintf(messages.ErrDirectInvoke, targetID, err))
		respondWithError(reqCtx, statusCode, msg)
		return
	}

	// 获取返回的响应数据，并设置返回数据到主调方
	invokev1.InternalMetadataToHTTPHeader(reqCtx, resp.Headers(), reqCtx.Response.Header.Set)
	contentType, body := resp.RawData()
	reqCtx.Response.Header.SetContentType(contentType)

	// Construct response
	statusCode := int(resp.Status().Code)
	if !resp.IsHTTPResponse() {
		statusCode = invokev1.HTTPStatusFromCode(codes.Code(statusCode))
		if statusCode != fasthttp.StatusOK {
			if body, err = invokev1.ProtobufToJSON(resp.Status()); err != nil {
				msg := NewErrorResponse("ERR_MALFORMED_RESPONSE", err.Error())
				respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
				return
			}
		}
	}
	respond(reqCtx, statusCode, body)
}

/*
 对于actor的处理，和前面其他资源处理是相同的，获取url参数，并根据传入参数获取对应的实例，然后根据这个实例提供的方法进行数据访问和操作，真正的业务逻辑都在实例提供的方法中，而dapr runtime http server onXXX handler接收外部请求，都只是获取请求数据，找到对应的实例对象，然后获取或者操作数据后，封装响应数据给主调方。没有特殊逻辑

 注意一点：单纯地对actor查询操作，包括事务批量查询，以及单个查询actor的状态数据，则需要调用IsActorHosted方法
 该方法用于校验actorType和actorID在dapr runtime actor实例的actorTable中是否存在，不存在表示当前系统中不存在该actor。可能已经销毁了
*/

// /v1.0/metdata
// onGetMetadata 获取dapr runtime http server存储有关业务元数据相关信息
// 包括appid、活跃actor数量、业务可以使用已注册使用的组件列表，以及外部传入的metdata元数据
//
// 外部传入的metdata，是通过onPutMetadata路由handler处理的，kv全部转换为string:string
func (a *api) onGetMetadata(reqCtx *fasthttp.RequestCtx) {
	temp := make(map[interface{}]interface{})

	// extendedMetadata 遍历元数据，并写入到temp作为响应数据的一部分
	// 这部分是用户自己的传入数据. 服务自身不提供任何元数据
	a.extendedMetadata.Range(func(key, value interface{}) bool {
		temp[key] = value
		return true
	})

	// 获取当前actor的actorId列表长度
	activeActorsCount := []actors.ActiveActorsCount{}
	if a.actor != nil {
		activeActorsCount = a.actor.GetActiveActorsCount(reqCtx)
	}

	// 获取dapr runtime已经注册的所有components，也就是可以当前microservice可以使用的数据
	registeredComponents := []registeredComponent{}

	for _, comp := range a.components {
		registeredComp := registeredComponent{
			Name:    comp.Name,
			Version: comp.Spec.Version,
			Type:    comp.Spec.Type,
		}
		registeredComponents = append(registeredComponents, registeredComp)
	}

	// 构建响应数据，并序列化后返回给主调
	mtd := metadata{
		ID:                   a.id,
		ActiveActorsCount:    activeActorsCount,
		Extended:             temp,
		RegisteredComponents: registeredComponents,
	}

	mtdBytes, err := a.json.Marshal(mtd)
	if err != nil {
		msg := NewErrorResponse("ERR_METADATA_GET", fmt.Sprintf(messages.ErrMetadataGet, err))
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		log.Debug(msg)
	} else {
		respondWithJSON(reqCtx, fasthttp.StatusOK, mtdBytes)
	}
}

// /v1.0/metadata/{key}
// onPutMetadata  获取路由中的key值，并取出请求body作为value，并存储到dapr runtime http server的extendmetadata内存中，随着microservice的销毁，这个数据也就不在了，一般提供给dapr runtime本地所在的microservice使用.
//
// 注意一点：request body的大小在microservice要有控制，不然内存可能会爆
func (a *api) onPutMetadata(reqCtx *fasthttp.RequestCtx) {
	key := fmt.Sprintf("%v", reqCtx.UserValue("key"))
	body := reqCtx.PostBody()
	a.extendedMetadata.Store(key, string(body))
	respondEmpty(reqCtx)
}
```

## 总结

本文为dapr runtime http server上篇，主要讲述了http server的路由构建过程、以及state/actor/secret/pubsub/direct_message/metadata/binding/health接收外部请求，并处理输入和输出，转发请求给第三方服务，包括components-contrib对应的第三方软件服务，或者microservices业务服务本身。转发本身没有什么业务逻辑，主要逻辑都在转发给的第三方服务。
