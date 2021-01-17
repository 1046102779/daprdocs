本文由三部分构成：
- 前言讲述actors、reminder和timer的相关概念；
- 分析actors的相关实现；
- 总结

通过本文，你可以了解到：
1. actors实现的机制，包括分布式应用下actors的服务发现和调用；
2. reminder和timer的异同。


## 前言

actor的模型在.net里面很成熟，dapr的actor就是来自于.net的orleans。

Actor模型作为一种用于处理并发计算的数学模型，它将Actor对象用作并发计算的通用单元，它也是一种重要的软件设计思想，它在架构、设计、实现以及组件之间的消息传递方面有着非常好的应用，也更好的发挥了多核计算机的潜力。通过创建新的Actor对象，可以在计算操作的生命周期中以抽象方式提高系统的分布性。

本文与[placement server: dapr源码系列一](https://github.com/1046102779/daprdocs/blob/master/pkg/placement.md)是强关联的。在k8s集群中，多个业务pod注入dapr sidecar，集群内多actor的分布式状态信息存储，是通过dapd-system命名空间下placement服务集群来完成。业务可以对一个要处理的大任务进行拆分，拆分成许多个小的子任务；或者对一个比较高的并发请求，进行请求的分发，拆分成一个个子请求；且每个子任务是独立不相关，是一个相对封闭的任务单元，也称为job。每个job封装了自己的数据和行为，不依赖外部。所以天然地支持和解决高并发问题。

目前actor支持的业务路由，主要是四类
1. state，对actor执行任务所需要数据的外部传递；
2. method，是对actor下的方法进行调用；也就是一个actor对象可以支持多个method任务处理；
3. reminders(定时提醒者), 这个是全局的reminder列表存储，是持久化存储。每隔一段时间daprd runtime就会调用执行一次任务；如果这个daprd runtime销毁后，这个当前被处理的reminders定期任务，会通过placement集群存储的hash一致性表分配到其他的daprd runtime上处理。后面详细介绍；
4. timers(定时器)， 这个不是全局的，也不会持久化存储；且只需当前daprd runtime有关系，当这个daprd runtime销毁后，就不会再发生执行任务的现象；

那么细化后要处理的请求，包括下面八种：
- PUT/POST actors/{actorType}/{actorId}/state
- GET/POST/DELETE/PUT actors/{actorType}/{actorId}/method/{method}
- GET actors/{actorType}/{actorId}/state/{key}
- POST/PUT actors/{actorType}/{actorId}/reminders/{name}
- POST/PUT actors/{actorType}/{actorId}/timers/{name}
- DELETE actors/{actorType}/{actorId}/reminders/{name}
- DELETE actors/{actorType}/{actorId}/timers/{name}
- GET actors/{actorType}/{actorId}/reminders/{name}

## 分析actors的相关实现

```golang
// Actors允许调用actor和actor的状态管理接口
type Actors interface {
    // GET/POST/DELETE/PUT actors/{actorType}/{actorId}/method/{method}
    // 访问提供{actor_type}和{method}服务的业务POD，进行任务处理
    Call(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMetho  dResponse, error)
    // Init初始化，daprd runtime actor状态信息存储到placement集群，并同时placement集群把分布式状态信息同步到各个actors, 同时获取reminders存储数据，然后根据placement集群的各个hash一致性表，校验reminder是否需要在正在初始化的daprd runtime actor中处理定时reminder任务。
    Init() error
    // Stop 停止daprd runtime actor工作。做一些收尾工作。
    Stop()
    // GetState 用于获取数据存储状态
    GetState(ctx context.Context, req *GetStateRequest) (*StateResponse, error)
    // TransactionalStateOperation 事务操作
    TransactionalStateOperation(ctx context.Context, req *TransactionalRequest) error
    // GetReminder 根据actor和name获取Reminder
    GetReminder(ctx context.Context, req *GetReminderRequest) (*Reminder, error)
    // CreateReminder 根据Reminder请求创建Reminder任务，持久化地定时执行任务
    CreateReminder(ctx context.Context, req *CreateReminderRequest) error
    // DeleteReminder 根据Reminder请求删除Reminder任务，并从持久化存储中删除该任务
    DeleteReminder(ctx context.Context, req *DeleteReminderRequest) error
    // CreateTimer 根据Timer请求创建Timer任务，并不做持久化，只存储到daprd runtime actor内存中，daprd销毁则timer任务也就停止，且不会被其他actor调度执行
    CreateTimer(ctx context.Context, req *CreateTimerRequest) error
    // DeleteTimer 根据Timer请求创建任务，只是删除内存中的Timer，并停止执行Timer任务
    DeleteTimer(ctx context.Context, req *DeleteTimerRequest) error
    // IsActorHosted 判断actor是否在本地
    IsActorHosted(ctx context.Context, req *ActorHostedRequest) bool
    // GetActiveActorsCount 获取当前daprd runtime actor正在执行actor任务的数量
    GetActiveActorsCount(ctx context.Context) []ActiveActorsCount
}

// actorsRuntime是Actors接口的一个实现类, 也是dapr产品中唯一的实现类。
type actorsRuntime struct {
      // appChannel 是daprd runtime接收外部请求后，把请求转发给该业务pod
      // 则这个appchannel是访问业务pod的grpc/http client
      appChannel          channel.AppChannel
      // store 是指actor的持久化存储，主要用来存储actor外部请求的对象数据，用于计算数据的来源
      // 或者reminder存储的actor_type所有数据。 对接的是component-contrib第三方软件实现类库
      store               state.Store
      // placement 上一节说的placement集群存储的所有actor分布式状态信息
      placement           *internal.ActorPlacement
      // grpcConnectionFn 用于获取grpc ClientConn连接
      grpcConnectionFn    func(address, id string, namespace string, skipTLS, recreateIfE  xists, enableSSL bool) (*grpc.ClientConn, error)
      // 创建actor runtime所需要的相关参数：dapr actor host, 业务pod的appid，placement集群地址列表
      // 业务pod支持的所有actor_types，及其他时间相关参数
      config              Config
      // actorsTable 用于创建actor_type和actor_id。
      // 当访问actor_type所在的actor runtime没有发现actor_id时，则会为client请求创建一个actor_type和actor_id的任务actor map， 不持久化
      actorsTable         *sync.Map
      // activeTimers 用于存储timers定时器列表，表示指定actor_type和actor_id和timer name下
      // 每隔一段时间执行一次任务，且不持久化。业务pod销毁后，其上的daprd runtime的所有timers也会被销毁。
      activeTimers        *sync.Map
      // activeTimersLock active timers并发锁
      activeTimersLock    *sync.RWMutex
      // activeReminders 是指placement集群上的哈希一致性表，分配到该dapr runtime actor的actor_type与actor_id列表
      //
      // 所以这里注意，当前reminder分配到该dapr runtime actor的，后面可能因为新加入的actor到集群中，
      // placement集群上的分布式状态存储中的hash一致性表把当前的actor中reminder分配到别的actor上了。
      // 所以需要每隔一段时间去看看当前的reminder是不是还在自己的dapr runtime actor上。
      activeReminders     *sync.Map
      remindersLock       *sync.RWMutex
      activeRemindersLock *sync.RWMutex
      // reminders 是map<actor_type>[]reminder, 每个actor_type存储相同的reminders。
      reminders           map[string][]Reminder
      // evaluation 用于校验actor是否处于繁忙状态
      evaluationLock      *sync.RWMutex
      evaluationBusy      bool
      evaluationChan      chan bool
      // appHealthy dapr runtime actor对业务POD的健康检查
      appHealthy          bool
      certChain           *dapr_credentials.CertChain
      // 分布式跟踪配置
      tracingSpec         config.TracingSpec
  }

// 对daprd runtime actor进行初始化，包括访问placement leader的初始化client，注册actor到placement分布式集群；
func (a *actorsRuntime) Init() error {
  // 首先daprd runtime actor启动，必须指定placement分布式集群地址列表信息
  //
  // 如果不传，则表示无法对actor信息进行分布式状态信息存储，直接报错
  if len(a.config.PlacementAddresses) == 0 {
      return errors.New("actors: couldn't connect to placement service: address is empty")
  }

  // HostedActorTypes 表示当前业务pod支持的actor_type类型数量为0
  // 则不应该进行actor的初始化创建动作
  if len(a.config.HostedActorTypes) > 0 {
      if a.store == nil {
          log.Warn("actors: state store must be present to initialize the actor runtime")
      }

      // 同时，对actor的存储，必须支持事务
      _, ok := a.store.(state.TransactionalStore)
      if !ok {
          return errors.New(incompatibleStateStore)
      }
  }

  // HostAddress Port是指dapr runtime的服务地址和端口，由环境变量指定DAPR_HOST_IP
  // 如果不指定，则随机生成
  hostname := fmt.Sprintf("%s:%d", a.config.HostAddress, a.config.Port)

  // afterTableUpdateFn 当集群内actor的信息发生变更时，也就是placement集群有信息进行三阶段提交推送每个actor时
  // 就会发生一次本dapr runtime actor的再平衡
  //
  // 原因是：新加入或者变更的actor，会从placement集群存储的分布式状态信息中的hash一致性表获取actor_type列表，然后再从store获取reminders列表后，再通过hash一致性表从actor_type和actor_id获取是不是本actor要处理这些reminder。
  // 另一个角度，就是本actor正在处理的，可能在发生一次整个集群的分布式状态信息更新时，这个reminder就不是本daprd runtime dapr处理了。所以需要清除，再平衡。
  // 还有一点，hash一致性表分配actor_type和actor_id任务到别的dapr runtime actor中处理后
  // 那本dapr runtime actor所在的业务pod需要删除当前actor_type和actor_id任务，不过这个必须等到actor不忙时。
  afterTableUpdateFn := func() {
      a.drainRebalancedActors()
      a.evaluateReminders()
  }
  // appHealthFn 校验当前业务pod的服务健康状态
  appHealthFn := func() bool { return a.appHealthy }

  // 创建actor存储整个集群的所有actor信息的actorplacement存储变量
  a.placement = internal.NewActorPlacement(
      a.config.PlacementAddresses, a.certChain,
      a.config.AppID, hostname, a.config.HostedActorTypes,
      appHealthFn,
      afterTableUpdateFn)

  // goroutine启动actor与placement的注册、通信和获取整个集群的分布式状态信息。
  // 用于路由和寻址
  go a.placement.Start()
  // startDeactivationTicker用于本dapr runtime actor清除长时间不用的actor。因为client可能不会或者忘记清除actor，导致大量actor泄漏
  a.startDeactivationTicker(a.config.ActorDeactivationScanInterval, a.config.ActorIdleTimeout)

  // goroutine 启动业务pod的定时健康检查, 如果业务pod不健康，则appHealthFn返回false。
  go a.startAppHealthCheck(
      health.WithFailureThreshold(4),
      health.WithInterval(5*time.Second),
      health.WithRequestTimeout(2*time.Second))

  return nil
}

// startAppHealthCheck 用于启动健康检查，在dapr中所有的健康检查路由都是/healthz
// 根据传入参数包括连续容错次数，健康检查周期和请求超时时间等
//
// 本业务pod服务提供actor服务，必须提供默认路由/healthz健康检查访问路由。这个可以由各个语言的sdk来保证
// 上面actor的启动健康检查，若连续4次都访问路由失败，则会通过channel回传不健康标志
func (a *actorsRuntime) startAppHealthCheck(opts ...health.Option) {
    if len(a.config.HostedActorTypes) == 0 {
        return
    }

    healthAddress := fmt.Sprintf("%s/healthz", a.appChannel.GetBaseAddress())
    ch := health.StartEndpointHealthCheck(healthAddress, opts...)
    for {
        // 连续访问设置的FailureThreshold失败次数，则appHealthy状态为false
        // 否则appHealthy为true，表示业务pod为健康状态
        a.appHealthy = <-ch
    }
}

// deactivateActor 删除actor.
// 包括两部分：1. 删除业务pod已经处理完成的<actor_type, actor_id>
// 2. 删除daprd runtime actor内存存储的actor_id
func (a *actorsRuntime) deactivateActor(actorType, actorID string) error {
    // 构建业务pod提供actor服务的路由：DELETE /actors/{actor_type}/{actor_id}
    req := invokev1.NewInvokeMethodRequest(fmt.Sprintf("actors/%s/%s", actorType, actorID))
    req.WithHTTPExtension(nethttp.MethodDelete, "")
    req.WithRawData(nil, invokev1.JSONContentType)

    // TODO Propagate context
    ctx := context.Background()
    // 通过访问业务POD服务的grpc或者http client，并把响应数据返回
    resp, err := a.appChannel.InvokeMethod(ctx, req)
    if err != nil {
        diag.DefaultMonitoring.ActorDeactivationFailed(actorType, "invoke")
        return err
    }

    if resp.Status().Code != nethttp.StatusOK {
        diag.DefaultMonitoring.ActorDeactivationFailed(actorType, fmt.Sprintf("status_c  ode_%d", resp.Status().Code))
        _, body := resp.RawData()
        return errors.Errorf("error from actor service: %s", string(body))
    }

    // 任务ID都是通过<actor_type>||<actor_id>构成的唯一ID，来进行存储的。并从dapr runtime actor中
    // 的actorsTable删除actor_id
    actorKey := a.constructCompositeKey(actorType, actorID)
    a.actorsTable.Delete(actorKey)
    diag.DefaultMonitoring.ActorDeactivated(actorType)
    log.Debugf("deactivated actor type=%s, id=%s\n", actorType, actorID)

    return nil
}

// startDeactivationTicker 启动并检查actor的闲置状态，并进行删除；
// 因为dapr没有给业务方提供删除actor_id的操作，主要原因有一个：
//
// 当新加入的dapr runtime actor，从placement集群中获取hash一致性列表，负载均衡时，很可能在dapr runtime actor A上处理的reminder，可能下一次是在dapr runtime actor B上运行。导致actor A泄漏.
func (a *actorsRuntime) startDeactivationTicker(interval, actorIdleTimeout time.Duration) {
      ticker := time.NewTicker(interval)
      // 起一个goroutine，然后每个interval时间，就会执行一次访问本daprd runtime actor存储的actorsTable任务，
      go func() {
          for t := range ticker.C {
              // 遍历actorsTable，执行本dapr runtime actor处理的所有actorid
              a.actorsTable.Range(func(key, value interface{}) bool {
                  actorInstance := value.(*actor)

                  // 如果actor当前正在执行, 则是active actor。不用删除
                  if actorInstance.isBusy() {
                      return true
                  }

                  // 如果actor上一次执行任务的时间到当前时间超过了actor的最大空闲时间
                  // 则说明actor不是active状态，需要清除该actor。
                  // 默认actorIdleTimeout为1小时
                  durationPassed := t.Sub(actorInstance.lastUsedTime)
                  if durationPassed >= actorIdleTimeout {
                      // 获取该actor的actor_type和actor_id
                      // 并通过grpc或者http client清除业务pod接收的子任务请求
                      // 并请求daprd runtime actor内存中actorsTable的actor
                      go func(actorKey string) {
                          actorType, actorID := a.getActorTypeAndIDFromKey(actorKey)
                          err := a.deactivateActor(actorType, actorID)
                          if err != nil {
                              log.Warnf("failed to deactivate actor %s: %s", actorKey, er  r)
                          }
                      }(key.(string))
                  }

                  return true
              })
          }
      }()
  }

// Call 表示接收外部请求，并把路由：/actors/{actor_type}/{actor_id}/methods/{method}转发给业务POD
//也就是actor_id下还有多个子任务。
//
// Call对传入的actor_id进行校验，看看actor_id是本地业务pod可以处理，还是其他同类型的actor_type的actor处理
func (a *actorsRuntime) Call(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
      a.placement.WaitUntilPlacementTableIsReady()

      // 获取请求的actor，并从actor内存存储的actorplacement中，并借助hash一致性表，负载均衡判断该actor是否在本pod处理
      actor := req.Actor()
      targetActorAddress, appID := a.placement.LookupActor(actor.GetActorType(), actor.GetActorId())
      if targetActorAddress == "" {
          return nil, errors.Errorf("error finding address for actor type %s with id %s",   actor.GetActorType(), actor.GetActorId())
      }

      var resp *invokev1.InvokeMethodResponse
      var err error

      // 比对本dapr runtime actor的服务地址，与hash一致性表给出的Host服务地址比对
      // 如果相同表示hash一致性表把该子任务分配给了本dapr runtime actor
      if a.isActorLocal(targetActorAddress, a.config.HostAddress, a.config.Port) {
          // 通过grpc或者client，转发给本业务pod服务，让业务pod处理该请求
          resp, err = a.callLocalActor(ctx, req)
      } else {
          // 否则, 本daprd runtime actor与目标daprd runtime actor建立grpc连接，相当于
          // 目标dapr runtime actor接收到外部请求，处理actor 调用流程。最终会走到目标业务pod服务处理任务流程
          resp, err = a.callRemoteActorWithRetry(ctx, retry.DefaultLinearRetryCount, retry.DefaultLinearBackoffInterval, a.callRemoteActor, targetActorAddress, appID, req)
      }

      if err != nil {
          return nil, err
      }
      return resp, nil
  }

// getOrCreateActor 存储actor_id实例并返回actor
// 从本地daprd runtime actor校验actor_id是否存在，如果不存在则添加，并返回actor对象
func (a *actorsRuntime) getOrCreateActor(actorType, actorID string) *actor {
    key := a.constructCompositeKey(actorType, actorID)

    // This avoids allocating multiple actor allocations by calling newActor
    // whenever actor is invoked. When storing actor key first, there is a chance to
    // call newActor, but this is trivial.
    val, ok := a.actorsTable.Load(key)
    if !ok {
        val, _ = a.actorsTable.LoadOrStore(key, newActor(actorType, actorID))
    }

    return val.(*actor)
}

// callLocalActor dapr runtime actor调用本地业务pod提供的actor服务。
// 这个请求可能来自内部其他dapr runtime actor的转发请求或者外部的直接调用
func (a *actorsRuntime) callLocalActor(ctx context.Context, req *invokev1.InvokeMethodR  equest) (*invokev1.InvokeMethodResponse, error) {
    // 获取actor，并获取actor_id的实例
    actorTypeID := req.Actor()
    act := a.getOrCreateActor(actorTypeID.GetActorType(), actorTypeID.GetActorId())
    // 这里要注意lock和unlock, 因为actor是并发安全的，所以业务pod处理actor请求处理任务时
    // 需要查看当前是否存在正在执行的任务，如果存在则阻塞，计数器：pendingActorCalls
    // lock锁：concurrencyLock
    err := act.lock()
    if err != nil {
        return nil, status.Error(codes.ResourceExhausted, err.Error())
    }
    // 当任务执行完成后，释放锁，然后阻塞的所有actor任务抢占该锁，获得锁的actor会请求本业务pod的actor任务路由
    defer act.unlock()

    // 业务pod的actor method请求路由PUT /actors/{actor_type}/{actor_id}/method/{method}
    req.Message().Method = fmt.Sprintf("actors/%s/%s/method/%s", actorTypeID.GetActorType(), actorTypeID.GetActorId(), req.Message().Method)
    // Original code overrides method with PUT. Why?
    if req.Message().GetHttpExtension() == nil {
        req.WithHTTPExtension(nethttp.MethodPut, "")
    } else {
        req.Message().HttpExtension.Verb = commonv1pb.HTTPExtension_PUT
    }
    // 通过业务pod服务提供的grpc或者http client访问业务pod服务，并进行业务服务处理。
    resp, err := a.appChannel.InvokeMethod(ctx, req)
    if err != nil {
        return nil, err
    }

    _, respData := resp.RawData()

    if resp.Status().Code != nethttp.StatusOK {
        return nil, errors.Errorf("error from actor service: %s", string(respData))
    }

    return resp, nil
}

// callRemoteActor 该方法表示要访问的{actor_type,actor_id, method}，并不在本业务pod上
// 这个不存在的原因有两个：
// 1. 该业务pod不支持处理请求带的actor_type类型任务；
// 2. 该业务pod支持请求带的actor_type类型任务，但是placement集群返回的分布式状态信息上的hash一致性表，并没有
// 把该actor_type任务交给该业务pod处理，很可能是因为有新进的actor或者删除actor后引起的负载均衡变化，导致
// 没有落到该业务pod上。
// 根据placement集群三阶段提交的整个集群分布式状态信息actor Host，就知道本dapr runtime actor要访问的目标地址
// 最终再转交给目标所在的业务pod执行任务
func (a *actorsRuntime) callRemoteActor(
      ctx context.Context,
      targetAddress, targetID string,
      req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
      // 下面就是获取目标dapr runtime actor grpc connection
      conn, err := a.grpcConnectionFn(targetAddress, targetID, a.config.Namespace, false,   false, false)
      if err != nil {
          return nil, err
      }

      span := diag_utils.SpanFromContext(ctx)
      ctx = diag.SpanContextToGRPCMetadata(ctx, span.SpanContext())
      client := internalv1pb.NewServiceInvocationClient(conn)
      // 通过grpc 访问CallActor API，使得目标dapr runtime actor接收到actor任务请求
      // 这个CallActor是pkg/grpc/api.go中的CallActor，它最终会本地actors实现的Call方法，也就是前面的Call访问
      resp, err := client.CallActor(ctx, req.Proto())
      if err != nil {
          return nil, err
      }

      return invokev1.InternalInvokeResponse(resp)
  }

// GetState 获取外部请求转发给dapr runtime actor的actor存储获取请求
// 获取actor的state是为了让这个业务pod执行actor任务时，存在数据传递。那么这个数据可以通过state方式获取
// 相当于这个actor的属性
func (a *actorsRuntime) GetState(ctx context.Context, req *GetStateRequest) (*StateResponse, error) {
    if a.store == nil {
        return nil, errors.New("actors: state store does not exist or incorrectly configured")
    }

    // metadata作为map存入：appid||actor_type||actor_id
    // 这个partitionKey作为是将数据分散到多个分区
    partitionKey := a.constructCompositeKey(a.config.AppID, req.ActorType, req.ActorID)
    metadata := map[string]string{metadataPartitionKey: partitionKey}

    // 因为state的存储是通过component-contrib提供的第三方存储服务进行管理的
    // 所以key必须具有唯一性，则key的构成：actor_type||actor_id||state key
    key := a.constructActorStateKey(req.ActorType, req.ActorID, req.Key)
    // 获取第三方存储的actor_id数据
    resp, err := a.store.Get(&state.GetRequest{
        Key:      key,
        Metadata: metadata,
    })
    if err != nil {
        return nil, err
    }

    return &StateResponse{
        Data: resp.Data,
    }, nil
}

// TransactionalStateOperation 用于事务提交，前面actor初始化时传入的store有一个校验，校验是不是支持事务state.TransactionalStore接口, 如果不支持直接报错
func (a *actorsRuntime) TransactionalStateOperation(ctx context.Context, req *TransactionalRequest) error {
    if a.store == nil {
        return errors.New("actors: state store does not exist or incorrectly configured  ")
    }
    // 构建partitionKey
    operations := []state.TransactionalStateOperation{}
    partitionKey := a.constructCompositeKey(a.config.AppID, req.ActorType, req.ActorID)
    metadata := map[string]string{metadataPartitionKey: partitionKey}

    // 一个事务的批量请求提交，遍历每个operation，进行数据协议转换
    // 然后添加到state.TransactionalStore接口的请求数据协议中。
    for _, o := range req.Operations {
        switch o.Operation {
        case Upsert:
            // 更新或者新增操作，则转换为目标协议数据
            var upsert TransactionalUpsert
            err := mapstructure.Decode(o.Request, &upsert)
            if err != nil {
                return err
            }
            // 构建key=actor_type||actor_id||operation key
            key := a.constructActorStateKey(req.ActorType, req.ActorID, upsert.Key)
            // 添加到目标协议数据operations中
            operations = append(operations, state.TransactionalStateOperation{
                Request: state.SetRequest{
                    Key:      key,
                    Value:    upsert.Value,
                    Metadata: metadata,
                },
                Operation: state.Upsert,
            })
        case Delete:
            // 删除操作，反序列化请求到TransactionDelete中
            var delete TransactionalDelete
            err := mapstructure.Decode(o.Request, &delete)
            if err != nil {
                return err
            }

            // 构建存储数据的key=actor_type||actor_id||operation key
            key := a.constructActorStateKey(req.ActorType, req.ActorID, delete.Key)
            // 添加到目标协议数据operations中
            operations = append(operations, state.TransactionalStateOperation{
                Request: state.DeleteRequest{
                    Key:      key,
                    Metadata: metadata,
                },
                Operation: state.Delete,
            })
        default:
            return errors.Errorf("operation type %s not supported", o.Operation)
        }
    }

    // 在actors Init初始化时，已经对store做过state.TransactionalStore接口实现断言
    // 表示actors存储的store是支持事务的
    transactionalStore, ok := a.store.(state.TransactionalStore)
    if !ok {
        return errors.New(incompatibleStateStore)
    }

    // 通过store的事务操作，提交批量数据存储操作，包括删除或者新增、修改等数据操作
    err := transactionalStore.Multi(&state.TransactionalStateRequest{
        Operations: operations,
        Metadata:   metadata,
    })
    return err
}

// drainRebalancedActors 重新平衡actors
// 因为新的dapr runtime actor的加入，或者旧的dapr runtime actor的销毁
// 都会使得placement集群存储的分布式状态信息，其上各类actor_type的hash一致性表，最终根据actor_id落在业务pod的负载均衡情况，会进行动态调整，所以这次正在运行的任务，可能在在下次就会分配到另一个dapr runtime actor的业务pod中处理任务
//
// 所以每当actorplacement存储数据发生变更时，就会进行一次更新，更新操作为:afterTableUpdateFn
func (a *actorsRuntime) drainRebalancedActors() {
    // visit all currently active actors
    var wg sync.WaitGroup

    // 每个actor_id使用一个goroutine，进行并发操作, 包括删除dapr runtime actor中内存存储的actor_id
    // 以及reminder、timer相关
    a.actorsTable.Range(func(key interface{}, value interface{}) bool {
        wg.Add(1)
        go func(key interface{}, value interface{}, wg *sync.WaitGroup) {
            defer wg.Done()
            actorKey := key.(string)
            actorType, actorID := a.getActorTypeAndIDFromKey(actorKey)
            // 判断actor_id是否会分配在本daprd runtime actor中处理
            //
            // 当actor_type存储的hash一致性存储数据不发生变化时，每次通过actorplacement的LookupActor方法
            // 获取到的Host是不会发生变化的
            // 如果dapr集群有新actor或者删除、修改actor，则可能会引起actor_id在集群内调度的调整
            address, _ := a.placement.LookupActor(actorType, actorID)
            // 判断hash一致性把actor_id分配到的dapr runtime actor host,是否依然是本地dapr runtime host
            // 如果不是，则说明上一次actor_id在本dapr runtime actor处理的任务，调度到别的dapr runtime actor处理了
            if address != "" && !a.isActorLocal(address, a.config.HostAddress, a.config.Port) {
                // 首先判断reminder是否有该actor_id的任务
                // 如果有，则删除本地缓存的reminder，以及删除state存储的远端reminder
                reminders := a.reminders[actorType]
                // 遍历本地dapr runtime actor的所有reminders，然后校验该actor上所有的actor_type和actor_id
                // 是否在reminder存在，如果存在则删除本地reminder
                // 这里的reminders最终都是从state远端的持久化存储中获取的。所以不需要直接在内存中删除它
                // 而是变更远端state存储的reminders后，再获取远端存储的state reminders，并缓存到actor的reminders中
                for _, r := range reminders {
                    if r.ActorType == actorType && r.ActorID == actorID {
                        reminderKey := a.constructCompositeKey(actorKey, r.Name)
                        stopChan, exists := a.activeReminders.Load(reminderKey)
                        if exists {
                            close(stopChan.(chan bool))
                            a.activeReminders.Delete(reminderKey)
                        }
                    }
                }

                actor := value.(*actor)
                // 如果DrainRebalancedActors为true，表示再平衡actor之前，
                // 需要等待当前阻塞的actor任务列表执行完成之后, 或者所有阻塞的任务执行超过1min，则也不再执行
                if a.config.DrainRebalancedActors {
                    // wait until actor isn't busy or timeout hits
                    if actor.isBusy() {
                        select {
                        case <-time.After(a.config.DrainOngoingCallTimeout):
                            break
                        case <-actor.channel():
                            // if a call comes in from the actor for state changes, tha  t's still allowed
                            break
                        }
                    }
                }

                // 等待业务pod中的actor_id实例任务单次执行完成后，删除业务pod中的actor_id任务
                // 同时在deactivateActor方法内也会删除dapr runtime actor内存存储的actorsTable中的actor_id
                // 这里是每隔300ms，再探测一次
                for {
                    if !actor.isBusy() {
                        err := a.deactivateActor(actorType, actorID)
                        if err != nil {
                            log.Warnf("failed to deactivate actor %s: %s", actorKey, er  r)
                        }
                        break
                    }
                    time.Sleep(time.Millisecond * 500)
                }
            }
        }(key, value, &wg)
        return true
    })
}

// evaluateReminders 主要用于reminders重新激活以及启动reminder
// 当dapr集群存储的分布式状态信息actors发生变更时,会触发afterTableUpdateFn动作
// 与前面的drainRebalancedActors方法属于同一个动作
// 在执行evaluateReminders过程中，所有的reminder操作都必须等待该操作完成，才能对reminder记性crud操作
func (a *actorsRuntime) evaluateReminders() {
    a.evaluationLock.Lock()
    defer a.evaluationLock.Unlock()

    // 重新整理reminders，并启动分配在本dapr中的reminder。
    // evaluationBusy设置为true
    a.evaluationBusy = true
    a.evaluationChan = make(chan bool)

    var wg sync.WaitGroup
    // 首先遍历获取本业务pod支持的所有actor_type任务类型
    // 针对每个类型获取当前存储的reminders列表，
    for _, t := range a.config.HostedActorTypes {
        // 通过components-contrib提供的state接口类，actors||actor_type作为key
        // rpc访问存储，获取reminders列表
        vals, err := a.getRemindersForActorType(t)
        if err != nil {
            log.Debugf("error getting reminders for actor type %s: %s", t, err)
        } else {
            // 然后再更新本地的reminders列表，所以本地daprd runtime actor是不会主动删除reminder的
            // 都是rpc同步远端存储到本地内存
            a.remindersLock.Lock()
            a.reminders[t] = vals
            a.remindersLock.Unlock()

            wg.Add(1)
            // 再对每类actor_type启动一个goroutine，然后遍历从远端存储获取的reminders列表
            // 对每个reminder进行校验，进行reminder任务启动，校验条件：
            // 1. 判断reminder是否由本业务pod负责执行任务；
            // 2. 判断当前reminder是否已经启动了。（这个通过activeReminders来判定）
            // 3. 如果没有启动，则加入到activeReminders中，并启动该reminder， 定时执行重复任务
            go func(wg *sync.WaitGroup, reminders []Reminder) {
                defer wg.Done()

                for i := range reminders {
                    r := reminders[i]
                    // 由hash一致性表根据负载均衡，分配reminder到指定的dapr runtime actor Host上
                    targetActorAddress, _ := a.placement.LookupActor(r.ActorType, r.Act  orID)
                    if targetActorAddress == "" {
                        continue
                    }

                    // 判断当前reminder是否被分配到本业务pod执行任务，这个是由hash一致性负载均衡决定
                    if a.isActorLocal(targetActorAddress, a.config.HostAddress, a.confi  g.Port) {
                        actorKey := a.constructCompositeKey(r.ActorType, r.ActorID)
                        reminderKey := a.constructCompositeKey(actorKey, r.Name)

                        // 判断当前reminder是否已经启动了，如果没有启动，则加入到activeReminder
                        // 且启动reminder定时任务。
                        _, exists := a.activeReminders.Load(reminderKey)

                        if !exists {
                            stop := make(chan bool)
                            a.activeReminders.Store(reminderKey, stop)
                            // 启动reminder定时任务
                            err := a.startReminder(&r, stop)
                            if err != nil {
                                log.Debugf("error starting reminder: %s", err)
                            }
                        }
                    }
                }
            }(&wg, vals)
        }
    }
    wg.Wait()
    close(a.evaluationChan)
    // 当分配给本dapr runtime actor的所有reminders都启动完成后，evaluationBusy设置为false
    a.evaluationBusy = false
}

// getReminderTrack 获取reminder最近一次触发执行任务的时间
func (a *actorsRuntime) getReminderTrack(actorKey, name string) (*ReminderTrack, error)   {
    // key: actor_type||actor_id||reminder name
    resp, err := a.store.Get(&state.GetRequest{
        Key: a.constructCompositeKey(actorKey, name),
    })
    if err != nil {
        return nil, err
    }

    // 反序列化解析数据，获取最近一次执行任务的时间
    var track ReminderTrack
    json.Unmarshal(resp.Data, &track)
    return &track, nil
}

// updateReminderTrack 当reminder触发一次执行任务后，就会更新state远端存储的reminder最近一次执行任务的时间
func (a *actorsRuntime) updateReminderTrack(actorKey, name string) error {
    track := ReminderTrack{
        LastFiredTime: time.Now().UTC().Format(time.RFC3339),
    }

    err := a.store.Set(&state.SetRequest{
        Key:   a.constructCompositeKey(actorKey, name),
        Value: track,
    })
    return err
}

// getUpcomingReminderInvokeTime 获取reminder下一次执行任务的时间
func (a *actorsRuntime) getUpcomingReminderInvokeTime(reminder *Reminder) (time.Time, error) {
      var nextInvokeTime time.Time

      // RegisteredTime为reminder的创建注册时间。
      registeredTime, err := time.Parse(time.RFC3339, reminder.RegisteredTime)
      if err != nil {
          return nextInvokeTime, errors.Wrap(err, "error parsing reminder registered time")
      }

      // 获取reminder首次启动任务的间隔时间
      dueTime, err := time.ParseDuration(reminder.DueTime)
      if err != nil {
          return nextInvokeTime, errors.Wrap(err, "error parsing reminder due time")
      }

      // 从state远端获取reminder最近一次执行任务的时间，首次添加时state存储中没有这个key数据
      key := a.constructCompositeKey(reminder.ActorType, reminder.ActorID)
      track, err := a.getReminderTrack(key, reminder.Name)
      if err != nil {
          return nextInvokeTime, errors.Wrap(err, "error getting reminder track")
      }

      // 如果track不为空，且最近一次执行任务的时间是存在的，则解析时间
      var lastFiredTime time.Time
      if track != nil && track.LastFiredTime != "" {
          lastFiredTime, err = time.Parse(time.RFC3339, track.LastFiredTime)
          if err != nil {
              return nextInvokeTime, errors.Wrap(err, "error parsing reminder last fired time")
          }
      }

      // 我觉得这块儿的代码有问题
      //  这块的整体逻辑是
      // 1. 如果首次执行reminder任务，则触发时间：registeredTime+DueTime
      // 2. 如果非首次执行reminder任务，则触发时间：lastFiredTime+period
      //
      // 这里要注意，针对第一种情况，如果period为空，则表示reminder只需要执行一次，然后立即删除该reminder
      // 如果首次执行任务之前的等待时间内，如果reminder通过CreateReminder修改该reminder的period或者dueTime
      // 比如：修改period为空，且新到的reminder执行任务的时间：dueTime+time.Now=time2, 而
      // 被覆盖的reminder还在等待该任务被执行，比如执行任务的等待时间为time1. 很可能time2<time1, 则导致后来的reminder任务先执行，然后再执行老的任务，这个是有问题的。
      //
      // 所以这里不应该出现：reminder第一次执行没有开始执行，后面来的reminder覆盖前者，且先于前者执行
      if reminder.Period != "" {
          period, err := time.ParseDuration(reminder.Period)
          if err != nil {
              return nextInvokeTime, errors.Wrap(err, "error parsing reminder period")
          }

          if !lastFiredTime.IsZero() {
              nextInvokeTime = lastFiredTime.Add(period)
          } else {
              // 第一次
              nextInvokeTime = registeredTime.Add(dueTime)
          }
      } else {
          if !lastFiredTime.IsZero() {
              nextInvokeTime = lastFiredTime.Add(dueTime)
          } else {
              nextInvokeTime = registeredTime.Add(dueTime)
          }
      }

      return nextInvokeTime, nil
  }

// startReminder 启动reminder任务
func (a *actorsRuntime) startReminder(reminder *Reminder, stopChannel chan bool) error   {
    // 根据传入的reminder和getUpcomingReminderInvokeTime获取reminder下一次的执行时间
    actorKey := a.constructCompositeKey(reminder.ActorType, reminder.ActorID)
    reminderKey := a.constructCompositeKey(actorKey, reminder.Name)
    nextInvokeTime, err := a.getUpcomingReminderInvokeTime(reminder)
    if err != nil {
        return err
    }

    // 启动goroutine，所以前面在getUpcomingReminderInvokeTime方法中说得nextInvokeTime逻辑
    // 是有问题的，因为都是goroutine等待执行状态，且下次一次执行时间因为reminder对dueTime的修改可能会乱序
    // 导致reminder的执行顺序错乱，这个是有问题的。
    go func(reminder *Reminder, stop chan bool) {
        now := time.Now().UTC()
        // 这个计算任务需要阻塞的时间，如果initialDuration小于等于0，则time.Sleep会立即返回
        initialDuration := nextInvokeTime.Sub(now)
        time.Sleep(initialDuration)

        // Check if reminder is still active
        select {
        case <-stop:
            log.Infof("reminder: %v with parameters: dueTime: %v, period: %v, data: %v   has been deleted.", reminderKey, reminder.DueTime, reminder.Period, reminder.Data)
            return
        default:
            break
        }

        // 访问本业务pod服务，立即执行任务
        err = a.executeReminder(reminder.ActorType, reminder.ActorID, reminder.DueTime,   reminder.Period, reminder.Name, reminder.Data)
        if err != nil {
            log.Errorf("error executing reminder: %s", err)
        }

        // 如果reminder的period不为空，则表示是个带有状态的cronjob过程
        if reminder.Period != "" {
            period, err := time.ParseDuration(reminder.Period)
            if err != nil {
                log.Errorf("error parsing reminder period: %s", err)
            }

            _, exists := a.activeReminders.Load(reminderKey)
            if !exists {
                log.Errorf("could not find active reminder with key: %s", reminderKey)
                return
            }

            // 然后再启动goroutine，每个period时间，就启动一次任务，如果reminder被删除后，
            // stop channel有事件通知，销毁该cronjob。
            t := a.configureTicker(period)
            go func(ticker *time.Ticker, actorType, actorID, reminder, dueTime, period   string,data interface{}) {
                for {
                    select {
                    case <-ticker.C:
                        err := a.executeReminder(actorType, actorID, dueTime, period, reminder, data)
                        if err != nil {
                            log.Debugf("error invoking reminder on actor %s: %s", a.con  structCompositeKey(actorType, actorID), err)
                        }
                    case <-stop:
                        log.Infof("reminder: %v with parameters: dueTime: %v, period: %  v, data: %v has been deleted.", reminderKey, dueTime, period, data)
                        return
                    }
                }
            }(t, reminder.ActorType, reminder.ActorID, reminder.Name, reminder.DueTime,   reminder.Period, reminder.Data)
        } else {
            // 如果period为空, 则表示初次执行reminder任务后，需要立即删除该任务，只执行一次，且不需要放到
            // state远端存储以及本地dapr activesReminders内存中
            err := a.DeleteReminder(context.TODO(), &DeleteReminderRequest{
                Name:      reminder.Name,
                ActorID:   reminder.ActorID,  
                ActorType: reminder.ActorType,
              })
              if err != nil {
                  log.Errorf("error deleting reminder: %s", err)
              }
          }
      }(reminder, stopChannel)

      return nil
  }

// executeReminder 立即执行reminder任务，
// 操作：本daprd runtime actor和本业务pod服务进行http rpc通信
// 最终发给业务pod服务的路由：/actors/actor_type/actor_id/method/remind/reminder name
func (a *actorsRuntime) executeReminder(actorType, actorID, dueTime, period, reminder string, data interface{}) error {
      r := ReminderResponse{
          DueTime: dueTime,
          Period:  period,
          Data:    data,
      }
      b, err := json.Marshal(&r)
      if err != nil {
          return err
      }

      log.Debugf("executing reminder %s for actor type %s with id %s", reminder, actorTyp  e, actorID)
      req := invokev1.NewInvokeMethodRequest(fmt.Sprintf("remind/%s", reminder))
      req.WithActor(actorType, actorID)
      req.WithRawData(b, invokev1.JSONContentType)

      _, err = a.callLocalActor(context.Background(), req)
      if err == nil {
          key := a.constructCompositeKey(actorType, actorID)
          // 任务执行成功后，则需要在state存储最后一次执行任务的时间lastFiredTime
          // 如果rpc 存储lastFiredTime失败，结果会有两种
          // 1. 当前state后端存储中lastFiredTime数据不在，则下一次执行时间，按照现在逻辑是registeredTime+dueTime, 则下一次执行时间非常可能是立即执行。 与期望比较大
          // 2. 当前state后端存储中的lastFiredTime数据已存在，则下一次执行时间，是lastFiredTime+period，则下一次执行任务的时间非常可能是立即执行。与期望比较大。
          // 我们不能随意改变reminder任务下一次的执行时间，这个是违反规则的
          // 我们应该主动报错
          a.updateReminderTrack(key, reminder)
      } else {
          log.Debugf("error execution of reminder %s for actor type %s with id %s: %s", r  eminder, actorType, actorID, err)
      }
      return err
  }

// CreateReminder daprd runtime actor接收外部用户的请求，最终转发到CreateReminder进行reminder的创建和修改
func (a *actorsRuntime) CreateReminder(ctx context.Context, req *CreateReminderRequest)   error {
      a.activeRemindersLock.Lock()
      defer a.activeRemindersLock.Unlock()
      r, exists := a.getReminder(req)
      if exists {
          // 首先判断当前reminder是否存在，如果存在则判断是否需要更新该reminder
          // 如果reminder发生变更，则需要先删除reminder
          if a.reminderRequiresUpdate(req, r) {
              err := a.DeleteReminder(ctx, &DeleteReminderRequest{
                  ActorID:   req.ActorID,
                  ActorType: req.ActorType,
                  Name:      req.Name,
              })
              if err != nil {
                  return err
              }
          } else {
              return nil
          }
      }

      // daprd runtime actor存储reminder到内存中
      actorKey := a.constructCompositeKey(req.ActorType, req.ActorID)
      reminderKey := a.constructCompositeKey(actorKey, req.Name)
      stop := make(chan bool)
      a.activeReminders.Store(reminderKey, stop)

      // 如果当前evaluationBusy为true，表示因为placement集群存储的分布式状态信息actors的修改
      // 导致placement集群通过三阶段提交，把分布式状态信息actors，推送给所有的dapr runtime actor
      // 然后再对reminder变化的reminder进行停止和重启
      if a.evaluationBusy {
          select {
          case <-time.After(time.Second * 5):
              return errors.New("error creating reminder: timed out after 5s")
          case <-a.evaluationChan:
              break
          }
      }

      reminder := Reminder{
          ActorID:        req.ActorID,
          ActorType:      req.ActorType,
          Name:           req.Name,
          Data:           req.Data,
          Period:         req.Period,
          DueTime:        req.DueTime,
          RegisteredTime: time.Now().UTC().Format(time.RFC3339),
      }

      // 从state远端的components-contrib对接的第三方存储中获取reminders
      reminders, err := a.getRemindersForActorType(req.ActorType)
      if err != nil {
          return err
      }

      reminders = append(reminders, reminder)

      // 然后再更新reminders到state远端存储中
      err = a.store.Set(&state.SetRequest{
          Key:   a.constructCompositeKey("actors", req.ActorType),
          Value: reminders,
      })
      if err != nil {
          return err
      }

      // 然后再更新本地的reminders信息
      a.remindersLock.Lock()
      a.reminders[req.ActorType] = reminders
      a.remindersLock.Unlock()

      // 最后启动remidner定时任务
      err = a.startReminder(&reminder, stop)
      if err != nil {
          return err
      }
}

// CreateTimer 创建Timer任务。
// 与Reminder差异在于，Timer不是持久化存储，它会随着业务pod的销毁而销毁，且不会被其他业务pod执行
// 而reminder是持久化存储，该业务pod销毁后，会被调度到其他业务pod上执行
func (a *actorsRuntime) CreateTimer(ctx context.Context, req *CreateTimerRequest) error   {
      a.activeTimersLock.Lock()
      defer a.activeTimersLock.Unlock()
      actorKey := a.constructCompositeKey(req.ActorType, req.ActorID)
      timerKey := a.constructCompositeKey(actorKey, req.Name)

      // 这里不明白，为啥actor_type||actor_id不存在，则就直接返回错误
      // 而不直接创建一个actor_type||actor_id, 难道所有的actor都需要通过Call进行method方法调用后才能生成吗？
      // 而reminder是不用创建的
      _, exists := a.actorsTable.Load(actorKey)
      if !exists {
          return errors.Errorf("can't create timer for actor %s: actor not activated", actorKey)
      }

      // 校验timerKey是否存在, 如果存在，则停止timer任务执行
      stopChan, exists := a.activeTimers.Load(timerKey)
      if exists {
          close(stopChan.(chan bool))
      }

      // 接下来，就是获取参数、解析参数然后执行timer定时任务
      d, err := time.ParseDuration(req.Period)
      if err != nil {
          return err
      }

      // 下面这段业务代码，主要是首次dueTime执行任务，然后再每隔period时间执行一次任务
      //
      // 这段逻辑是有问题的。因为第二次执行Timer任务时，应该要等待第一次任务执行完成后才开始计算。
      // 而不是period和dueTime同时计时
      t := a.configureTicker(d)
      stop := make(chan bool, 1)
      a.activeTimers.Store(timerKey, stop)


      go func(ticker *time.Ticker, stop chan (bool), actorType, actorID, name, dueTime, period, callback string, data interface{}) {
          // 首次执行Timer任务，等待dueTime时间
          if dueTime != "" {
              d, err := time.ParseDuration(dueTime)
              if err == nil {
                  time.Sleep(d)
              }
          }

          // Check if timer is still active
          select {
          case <-stop:
              log.Infof("Time: %v with parameters: DueTime: %v, Period: %v, Data: %v has been deleted.", timerKey, req.DueTime, req.Period, req.Data)
              return
          default:
              break
          }

          // daprd runtime actor访问本业务pod的timer服务
          // 访问业务pod提供的固定路由： actors/actor_type/actor_type/method/timer/timer name
          err := a.executeTimer(actorType, actorID, name, dueTime, period, callback, data  )
          if err != nil {
              log.Debugf("error invoking timer on actor %s: %s", actorKey, err)
          }

          actorKey := a.constructCompositeKey(actorType, actorID)

          // 然后每隔period时间，执行一次Timer任务
          for {
              select {
              case <-ticker.C:
                  _, exists := a.actorsTable.Load(actorKey)
                  if exists {
                      err := a.executeTimer(actorType, actorID, name, dueTime, period, ca  llback, data)
                      if err != nil {
                          log.Debugf("error invoking timer on actor %s: %s", actorKey, er  r)
                      }
                  } else {
                  //如果不存在，则表示Timer只执行一次
                      a.DeleteTimer(ctx, &DeleteTimerRequest{
                          Name:      name,
                          ActorID:   actorID,
                          ActorType: actorType,
                      })
                  }
              case <-stop:
                  return
              }
          }
      }(t, stop, req.ActorType, req.ActorID, req.Name, req.DueTime, req.Period, req.Callback, req.Data)
      return nil
  }

// executeTimer 立即执行Timer定时任务，daprd runtime actor访问本业务pod提供的Timer服务
// 让业务逻辑执行一次cronjob。
func (a *actorsRuntime) executeTimer(actorType, actorID, name, dueTime, period, callback string, data  interface{}) error {
    t := TimerResponse{
        Callback: callback,
        Data:     data,
        DueTime:  dueTime,
        Period:   period,
    }
    b, err := json.Marshal(&t)
    if err != nil {
        return err
    }

    // 最终访问业务pod的业务路由：
    // actors/actor_type/actor_id/method/timer/timer name
    log.Debugf("executing timer %s for actor type %s with id %s", name, actorType, actorID)
    req := invokev1.NewInvokeMethodRequest(fmt.Sprintf("timer/%s", name))
    req.WithActor(actorType, actorID)
    req.WithRawData(b, invokev1.JSONContentType)
    // 调用本业务pod，提供的http服务
    _, err = a.callLocalActor(context.Background(), req)
    if err != nil {
        log.Debugf("error execution of timer %s for actor type %s with id %s: %s", name, actorType, ac  torID, err)
    }
    return err
}

// getRemindersForActorType 根据actors/actor_type 获取state远端存储所有的reminders列表
func (a *actorsRuntime) getRemindersForActorType(actorType string) ([]Reminder, error) {
      key := a.constructCompositeKey("actors", actorType)
      resp, err := a.store.Get(&state.GetRequest{
          Key: key,
      })
      if err != nil {
          return nil, err
      }

      var reminders []Reminder
      json.Unmarshal(resp.Data, &reminders)
      return reminders, nil
}

// DeleteReminder 删除reminder操作, 根据actor_type、actor_id和reminder name删除指定reminder
func (a *actorsRuntime) DeleteReminder(ctx context.Context, req *DeleteReminderRequest) error {
    // 首先校验reminder是否处于调整状态，如果是
    // 则需要等待reminder调整结束。设置一个默认超时时间
    if a.evaluationBusy {
        select {
        case <-time.After(time.Second * 5):
            return errors.New("error creating reminder: timed out after 5s")
        case <-a.evaluationChan:
            break
        }
    }

    key := a.constructCompositeKey("actors", req.ActorType)
    actorKey := a.constructCompositeKey(req.ActorType, req.ActorID)
    reminderKey := a.constructCompositeKey(actorKey, req.Name)

    // 根据actor_type||actor_id||reminder name key，查找是否存在reminder
    // 如果存在，则在内存中删除这个reminder
    stop, exists := a.activeReminders.Load(reminderKey)
    if exists {
        log.Infof("Found reminder with key: %v. Deleting reminder", reminderKey)
        close(stop.(chan bool))
        a.activeReminders.Delete(reminderKey)
    }

    // 同时需要把state远端存储的相关reminder汇总数据，进行更新操作
    //
    // 首先删除远端actors||actor_type 存储key，获取reminders列表，然后找到要删除的reminder
    // 再做更新reminder列表操作
    reminders, err := a.getRemindersForActorType(req.ActorType)
    if err != nil {
        return err
    }
    for i := len(reminders) - 1; i >= 0; i-- {
        if reminders[i].ActorType == req.ActorType && reminders[i].ActorID == req.ActorID && reminders  [i].Name == req.Name {
            reminders = append(reminders[:i], reminders[i+1:]...)
        }
    }

    err = a.store.Set(&state.SetRequest{
        Key:   key,
        Value: reminders,
    })
    if err != nil {
        return err
    }

    // 把更新后相同的actor_type的reminder列表，存储到reminders列表中
    a.remindersLock.Lock()
    a.reminders[req.ActorType] = reminders
    a.remindersLock.Unlock()

    // 最后删除actor_type||actor_id||reminder name key state远端存储的该reminder数据
    err = a.store.Delete(&state.DeleteRequest{
        Key: reminderKey,
    })
    if err != nil {
        return err
    }

    return nil
}

// GetReminder 根据请求actor_type||actor_id||reminder name获得reminder
//
// 这个方法一般用于在placement集群存储的分布式状态信息actors发生变更后，就可能会因为hash一致性表负载均衡策略，二会引起reminder的调度调整
// 然后重新整理reminder列表和加载启动reminder
func (a *actorsRuntime) GetReminder(ctx context.Context, req *GetReminderRequest) (*Reminder,error) {
      // 根据actors||actor_type作为key，获取state远端的reminder列表
      reminders, err := a.getRemindersForActorType(req.ActorType)
      if err != nil {
          return nil, err
      }

      // 找到传入的reminder是否存在
      for _, r := range reminders {
          if r.ActorID == req.ActorID && r.Name == req.Name {
              return &Reminder{
                  Data:    r.Data,
                  DueTime: r.DueTime,
                  Period:  r.Period,
              }, nil
          }
      }
      return nil, nil
  }

// DeleteTimer 因为Timer定时任务，只参与本daprd runtime actor的cronjob。不持久化，所以是内存存储。
// 随着业务pod的销毁，而销毁
func (a *actorsRuntime) DeleteTimer(ctx context.Context, req *DeleteTimerRequest) error {
      actorKey := a.constructCompositeKey(req.ActorType, req.ActorID)
      timerKey := a.constructCompositeKey(actorKey, req.Name)

      // 所以只在daprd runtime actor内存中删除actor_type||actor_id||timer name
      stopChan, exists := a.activeTimers.Load(timerKey)
      if exists {
          close(stopChan.(chan bool))
          a.activeTimers.Delete(timerKey)
      }

      return nil
  }

// GetActiveActorsCount 根据本业务pod支持的所有actor_type类型和当前业务pod已接收的actorsTable
// 进行actor_id实例的计算
func (a *actorsRuntime) GetActiveActorsCount(ctx context.Context) []ActiveActorsCount {
      actorCountMap := map[string]int{}
      // 本业务pod支持的所有actor_type，把actor_types列表转换成map结构
      // 先以map方式记录业务pod支持的各个actor_type
      for _, actorType := range a.config.HostedActorTypes {
          actorCountMap[actorType] = 0
      }
      // 然后遍历所有的map key。针对每个key把actor_type||actor_id，分离进行统计
      // 获取每个actor_type当前业务pod处理的actor_id实例数量
      a.actorsTable.Range(func(key, value interface{}) bool {
          actorType, _ := a.getActorTypeAndIDFromKey(key.(string))
          actorCountMap[actorType]++
          return true
      })

      // 最后再把actorCountMap转换成struct实例{actor_type, count}
      activeActorsCount := []ActiveActorsCount{}
      for actorType, count := range actorCountMap {
          activeActorsCount = append(activeActorsCount, ActiveActorsCount{Type: actorType, Count: count}  )
      }

      return activeActorsCount
  }
```

## 总结

以上就是dapr runtime actors的所有内容，通过与上一篇[placement server: dapr源码系列一](https://github.com/1046102779/daprdocs/blob/master/pkg/placement.md)文章相结合，你就知道的dapr产品所有关于actor、placement整体架构，了解了内部运行原理。
