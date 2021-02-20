messaging包用于dapr runtime之间的RPC服务调用， 它通过[nameresolution](https://github.com/dapr/components-contrib/tree/master/nameresolution)名字服务来实现服务发现，并进行grpc服务调用，然后请求到达被调dapr runtime后，再通过http/grpc来访问本地的应用端口服务，实现microservice rpc服务调用。

接下来就对messaging包实现服务之间的调用源码分析

## rpc请求pb协议

service invocation服务调用PB协议定义如下：

> github.com/dapr/dapr/dapr/proto/common/v1
> github.com/dapr/dapr/dapr/proto/internals/v1

```pb
// InvokeRequest 服务调用请求的数据
message InvokeRequest {
  // 请求目标方法, 类似于路由
  string method = 1;

  // 任意类数据，这里是指数据流，里面含有数据流的数据类型
  google.protobuf.Any data = 2;

  // 数据类型的协议类型：application/json application/grpc+xxx
  string content_type = 3;

  // http扩展字段，表示http请求方法GET/POST/..., 以及queryString
  HTTPExtension http_extension = 4;
}

// InvokeResponse 服务调用响应结果
message InvokeResponse {
  // 响应数据流，也包括数据类型
  google.protobuf.Any data = 1;

  // 数据类型的协议类型： application/json application/grpc+xxx
  string content_type = 2;
}
```

下面为RPC方法调用接口定义:

```pb
// ServiceInvocation daprd runtime服务RPC调用接口
// 两个方法：
// 1. CallActor方法：dapr actor之间的RPC，它有一套自己独立的服务发现和服务注册，并通过dapr-system命名空间下的dapr-placement服务提供支持
// 2. CallLocal方法：是指dapr runtime之间服务直接调用，它通过contrib-components中的nameresolution来实现服务发现。目前这个CallLocal只是说明本地调用，如果改为Call更加好一点
service ServiceInvocation {
  // CallActor actor之间的rpc调用
  rpc CallActor (InternalInvokeRequest) returns (InternalInvokeResponse) {}

  // service invocation 服务RPC之间的调用
  rpc CallLocal (InternalInvokeRequest) returns (InternalInvokeResponse) {}
}

// Actor 包括actor_type和actor_id，代表一个actor实例对象
message Actor {
  // Required. The type of actor.
  string actor_type = 1;

  // Required. The ID of actor type (actor_type)
  string actor_id = 2;
}

// InternalInvokeRequest 封装一个rpc请求，并携带版本信息、metadata和可能存在的actor信息
message InternalInvokeRequest {
  // Required. The version of Dapr runtime API.
  APIVersion ver = 1;

  // Required. metadata holds caller's HTTP headers or gRPC metadata.
  map<string, ListStringValue> metadata = 2;

  // Required. message including caller's invocation request.
  common.v1.InvokeRequest message = 3;

  // Actor type and id. This field is used only for
  // actor service invocation.
  Actor actor = 4;
}

// InternalInvokeResponse 封装一个rpc响应信息，并协议响应码和错误信息，并协议headers、trailers
message InternalInvokeResponse {
  // Required. HTTP/gRPC status.
  Status status = 1;

  // Required. The app callback response headers.
  map<string, ListStringValue> headers = 2;

  // App callback response trailers.
  // This will be used only for gRPC app callback
  map<string, ListStringValue> trailers = 3;

  // Callee's invocation response message.
  common.v1.InvokeResponse message = 4;
}

// ListStringValue metadata中的key值列表
message ListStringValue {
  // The array of string.
  repeated string values = 1;
}
```

在`github.com/dapr/dapr/pkg/messaging/v1`下的代码都是针对RPC请求的参数，以及RPC响应参数的设置和获取，以及数据协议转换等, 比较简单

## rpc服务直接调用

```golang
// messageClientConnection 函数类型用于创建一个grpc connection。这个函数实现类，则会实现ServiceInvocation接口
type messageClientConnection func(address, id string, namespace string, skipTLS, recrea  teIfExists, enableSSL bool) (*grpc.ClientConn, error)

// DirectMessaging RPC调用封装接口，对daprd runtime提供RPC服务调用
type DirectMessaging interface {
    Invoke(ctx context.Context, targetAppID string, req *invokev1.InvokeMethodRequest)   (*invokev1.InvokeMethodResponse, error)
}

// directMessaging daprd runtime之间的直接调用
type directMessaging struct {
    // appChannel 指同一个pod中的daprd runtime和microservice业务容器直接通信
    appChannel          channel.AppChannel
    // 主调daprd runtime与被调daprd runtime建立grpc connection连接
    connectionCreatorFn messageClientConnection
    // 主调appid
    appID               string
    // 模式：k8s或者standalone
    mode                modes.DaprMode
    // daprd runtime之间的grpc port
    grpcPort            int
    // daprd runtime所在pod的命名空间
    namespace           string
    // 被调daprd runtime的服务发现，就是靠着contrib-components中的nameresolution名字服务来实现
    resolver            nr.Resolver
    // 分布式跟踪配置
    tracingSpec         config.TracingSpec
    // daprd runtime所在的服务地址
    hostAddress         string
    // daprd runtime所在的服务主机名称
    hostName            string
    // 最大请求数据包，单位：MB
    maxRequestBodySize  int
}

// nameresolution名字服务发现的数据返回对象
type remoteApp struct {
    id        string
    namespace string
    address   string
}

// NewDirectMessaging 创建一个direct message接口对象
func NewDirectMessaging(
    appID, namespace string,
    port int, mode modes.DaprMode,
    appChannel channel.AppChannel,
    clientConnFn messageClientConnection,
    resolver nr.Resolver,
    tracingSpec config.TracingSpec, maxRequestBodySize int) DirectMessaging {
    hAddr, _ := utils.GetHostAddress()
    hName, _ := os.Hostname()
    return &directMessaging{
        appChannel:          appChannel,
        connectionCreatorFn: clientConnFn,
        appID:               appID,
        mode:                mode,
        grpcPort:            port,
        namespace:           namespace,
        resolver:            resolver,
        tracingSpec:         tracingSpec,
        hostAddress:         hAddr,
        hostName:            hName,
        maxRequestBodySize:  maxRequestBodySize,
    }
}

// Invoke 用于daprd runtime发起一个rpc直接调用，并返回rpc请求的响应数据.
func (d *directMessaging) Invoke(ctx context.Context, targetAppID string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  // 通过参数targetAppId，被调appid，通过服务发现来找到远端daprd runtime的地址信息
	app, err := d.getRemoteApp(targetAppID)
	if err != nil {
		return nil, err
	}

  // 并校验被调daprd runtime目标服务的appid和命名空间，是否和本daprd runtime一致
  // 如果一致，则表示这个调用为本地调用，直接通过appChannel访问本POD中的业务microservice容器服务
	if app.id == d.appID && app.namespace == d.namespace {
		return d.invokeLocal(ctx, req)
	}
  // 否则，发起远程调用，则首先通过daprd runtime访问被调的daprd runtime服务
	return d.invokeWithRetry(ctx, retry.DefaultLinearRetryCount, retry.DefaultLinearBackoffInterval, app, d.invokeRemote, req)
}

// requestAppIDAndNamespace 解析目标appid，获取目标服务的appid和namespace
func (d *directMessaging) requestAppIDAndNamespace(targetAppID string) (string, string, error) {
	items := strings.Split(targetAppID, ".")
	if len(items) == 1 {
    // 如果目标appid不携带namespace，则表示目标服务与主调服务在同一个命名空间
		return targetAppID, d.namespace, nil
	} else if len(items) == 2 {
		return items[0], items[1], nil
	} else {
		return "", "", errors.Errorf("invalid app id %s", targetAppID)
	}
}

// invokeWithRetry 发起daprd runtime远程调用，是daprd runtime之间的RPC服务调用
func (d *directMessaging) invokeWithRetry(
	ctx context.Context,
	numRetries int,
	backoffInterval time.Duration,
	app remoteApp,
	fn func(ctx context.Context, appID, namespace, appAddress string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error),
	req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  // 重试和退避策略，fn是指invokeRemote
	for i := 0; i < numRetries; i++ {
    // fn是指invokeRemote方法调用，它用于创建grpc client connection，并通过
    // 上面pb协议定义的ServiceInvocation接口CallLocal进行rpc服务调用
		resp, err := fn(ctx, app.id, app.namespace, app.address, req)
    // 如果rpc调用成功，则直接返回
		if err == nil {
			return resp, nil
		}
    // 否则执行退避策略和重试策略
		time.Sleep(backoffInterval)

		code := status.Code(err)
    // 如果返回的错误，不可达或者没有授权访问，则直接试图发起grpc client连接建立
    // 如果失败，直接返回失败错误
    // 否则如果为其他错误，则继续重试
		if code == codes.Unavailable || code == codes.Unauthenticated {
			_, connerr := d.connectionCreatorFn(app.address, app.id, app.namespace, false, true, false)
			if connerr != nil {
				return nil, connerr
			}
			continue
		}
		return resp, err
	}
	return nil, errors.Errorf("failed to invoke target %s after %v retries", app.id, numRetries)
}

// invokeLocal 表示是业务POD中的daprd runtime访问本地的microservice
// 也就是一个pod中的两个容器之间的服务调用：daprd runtime调用本地业务服务
func (d *directMessaging) invokeLocal(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	if d.appChannel == nil {
		return nil, errors.New("cannot invoke local endpoint: app channel not initialized")
	}

  // 通过directMessaging提供的appChannel进行http或者grpc本地服务调用
	return d.appChannel.InvokeMethod(ctx, req)
}

// invokeRemote 一次daprd runtime服务与远端的daprd runtime之间的RPC直接调用
func (d *directMessaging) invokeRemote(ctx context.Context, appID, namespace, appAddress string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  // connectionCreatorFn函数创建与远端daprd runtime之间的grpc client connection
  // 并返回这个连接，这个connectionCreatorFn返回的connection实现了ServiceInvocation接口
  //
  // 目前这个connectionCreatorFn是指grpc中的manager的GetGRPCConnection方法
	conn, err := d.connectionCreatorFn(appAddress, appID, namespace, false, false, false)
	if err != nil {
		return nil, err
	}

  // 参数填充，包括tracing、appid和真正用于请求业务服务microservice的请求参数
	span := diag_utils.SpanFromContext(ctx)
	ctx = diag.SpanContextToGRPCMetadata(ctx, span.SpanContext())

	d.addForwardedHeadersToMetadata(req)
	d.addDestinationAppIDHeaderToMetadata(appID, req)

	clientV1 := internalv1pb.NewServiceInvocationClient(conn)

	var opts []grpc.CallOption
	opts = append(opts, grpc.MaxCallRecvMsgSize(d.maxRequestBodySize*1024*1024), grpc.MaxCallSendMsgSize(d.maxRequestBodySize*1024*1024))

  // 最终发起grpc client 远程调用，访问主调
	resp, err := clientV1.CallLocal(ctx, req.Proto(), opts...)
	if err != nil {
		return nil, err
	}

  // 返回响应信息
	return invokev1.InternalInvokeResponse(resp)
}

// addDestinationAppIDHeaderToMetadata 请求参数InvokeMethodRequest的metadata填充目标appid参数
func (d *directMessaging) addDestinationAppIDHeaderToMetadata(appID string, req *invokev1.InvokeMethodRequest) {
	req.Metadata()[invokev1.DestinationIDHeader] = &internalv1pb.ListStringValue{
		Values: []string{appID},
	}
}

// addForwardedHeadersToMetadata 请求参数InvokeMethodRequest填充主调的地址和hostname等信息到metdata中
func (d *directMessaging) addForwardedHeadersToMetadata(req *invokev1.InvokeMethodRequest) {
	metadata := req.Metadata()

	var forwardedHeaderValue string

	if d.hostAddress != "" {
		// Add X-Forwarded-For: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For
		metadata[fasthttp.HeaderXForwardedFor] = &internalv1pb.ListStringValue{
			Values: []string{d.hostAddress},
		}

		forwardedHeaderValue += "for=" + d.hostAddress + ";by=" + d.hostAddress + ";"
	}

	if d.hostName != "" {
		// Add X-Forwarded-Host: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Host
		metadata[fasthttp.HeaderXForwardedHost] = &internalv1pb.ListStringValue{
			Values: []string{d.hostName},
		}

		forwardedHeaderValue += "host=" + d.hostName
	}

	// Add Forwarded header: https://tools.ietf.org/html/rfc7239
	metadata[fasthttp.HeaderForwarded] = &internalv1pb.ListStringValue{
		Values: []string{forwardedHeaderValue},
	}
}

// getRemoteApp 通过目标appid和nameresolution，进行服务发现并返回远端daprd runtime的地址信息
func (d *directMessaging) getRemoteApp(appID string) (remoteApp, error) {
  // 通过目标appid获取appid和appid所在的namespace
	id, namespace, err := d.requestAppIDAndNamespace(appID)
	if err != nil {
		return remoteApp{}, err
	}

  // 并把目标appid和所在的namespace，以及daprd runtime作为输入参数
  // 通过nameresolution名字服务找到daprd runtime目标服务地址
	request := nr.ResolveRequest{ID: id, Namespace: namespace, Port: d.grpcPort}
	address, err := d.resolver.ResolveID(request)
	if err != nil {
		return remoteApp{}, err
	}

  // 并发响应结果返回，包括目标appid，namespace和address
	return remoteApp{
		namespace: namespace,
		id:        id,
		address:   address,
	}, nil
}
```

通过messaging包，我们就可以实现集群内部微服务之间的RPC直接调用，最关键的地方在于nameresolution名字服务, 目前在contrib-components中的nameresolution支持两种名字服务，一种为k8s, 另一个种为mdns，其他暂时不支持。

1. k8s的nameresolution：{appId}-dapr.{namespace}.svc.cluster.local:{grpcPort}
2. mdns基于局域网的服务发现

另一方面directMessaging对象中的行为Invoke方法，调用的上游在`github.com/dapr/dapr/pkg/grpc`和`github.com/dapr/dapr/pkg/http`中。

针对grpc和http两种调用：
1. daprd runtime grpc对外提供的服务方法API为：**InvokeService**;
2. daprd runtime http对外提供的服务方法API为: **onDirectMessage**, 也就是路由为：`invoke/{id}/method/{method:*}`中

这样外部用户或者内部业务微服务，就可以通过网关或者自身的业务POD内部的daprd runtime容器，就可以在daprd runtime服务中找到目标业务POD服务，并进行rpc直接调用，并返回响应数据给主调方
