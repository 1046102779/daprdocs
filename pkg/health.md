健康检查在dapr中以一个独立的服务模块存在，只需要进行监控包引入和 初始化，则就可以启动服务的健康检查。这个health包提供了server端和client端。

## 健康检查服务端

业务POD、或者dapr内部服务需要使用dapr health健康检查服务包，则需要创建健康检查goroutine服务，这里分三步：
1. 通过NewServer API，创建健康检查服务实例；
2. 通过Run启动健康检查端口服务，并对外提供**/healthz**健康检查路由服务；
3. 通过Ready和NotReady两个API，来设置引入该health包的服务健康状态；

```golang
// Server 表示健康检查服务接口，启动健康检查服务和设置服务健康状态
type Server interface {
  Run(context.Context, int) error
  Ready()
  NotReady()
}

// server 表示Server接口的一个实现类
type server struct {
  ready bool
  log   logger.Logger
}

// NewServer 创建一个健康检查实例server
//
// 健康检查的日志输出与引入该health包的日志保持一致
func NewServer(log logger.Logger) Server {
  return &server{
      log: log,
  }
}

// Ready 设置服务的健康检查状态为健康状态
func (s *server) Ready() {
  s.ready = true
}

// NotReady 设置服务的健康检查状态为不健康状态
func (s *server) NotReady() {
  s.ready = false
}

// Run 启动健康检查fasthttp端口服务, 并通过上一节的signals包，对context进行上下游整个链路控制
//
// 当context接收到进程退出信号时，则上下文ctx会通过Done来捕获退出信号，然后再优雅地关闭健康检查服务
func (s *server) Run(ctx context.Context, port int) error {
  // 创建健康检查服务，并提供/healthz路由服务，来告诉主调，被掉是否健康
  router := http.NewServeMux()
  router.Handle("/healthz", s.healthz())

  srv := &http.Server{
      Addr:    fmt.Sprintf(":%d", port),
      Handler: router,
  }

  doneCh := make(chan struct{})

  // 启动goroutine， 当上游context关闭(进程退出)， 则健康检查服务通过该信号优雅退出
  //
  // 另一个角度，当健康检查服务异常退出时，则可以通过close channel来优雅关闭goroutine
  go func() {
      select {
      case <-ctx.Done():
          s.log.Info("Healthz server is shutting down")
          shutdownCtx, cancel := context.WithTimeout(
              context.Background(),
              time.Second*5,
          )
          defer cancel()
          srv.Shutdown(shutdownCtx) // nolint: errcheck
      case <-doneCh:
      }
    }()

    s.log.Infof("Healthz server is listening on %s", srv.Addr)
    err := srv.ListenAndServe()
    if err != http.ErrServerClosed {
        s.log.Errorf("Healthz server error: %s", err)
    }
    close(doneCh)
    return err
}

// healthz 是健康检查服务的路由/healthz API，响应给主服务，告知服务健康状态
//
// 这种一般都是通过goroutine实现，当ready为true表示，主服务当前状态为健康，否则不健康，不能提供正常的服务
// 健康时通过http 200返回，不健康时通过http 503返回
func (s *server) healthz() http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        var status int
        if s.ready {
            status = http.StatusOK
        } else {
            status = http.StatusServiceUnavailable
        }
        w.WriteHeader(status)
    })
}
```

### 健康检查客户端

健康检查客户端, 也是主服务通过goroutine来实时获取服务健康状态， 一个服务是否健康，需要设置一些条件。

比如，健康检查探测周期，连续探测失败次数，等待健康检查服务启动的初始化时间, 以及请求健康检查服务的超时时间等

```golang
const (
  // 默认参数值设置，也就是上面说的四个参数
  // 再加上健康检查http成功状态码, 默认为200
  //
  // 如果默认参数，那么一个服务不健康的话，需要2*5=10s，也就是10s的健康探测都是失败状态，则为不健康状态
  initialDelay      = time.Second * 1
  failureThreshold  = 2
  requestTimeout    = time.Second * 2
  interval          = time.Second * 5
  successStatusCode = 200
)

// Option 健康检查客户端函数参数赋值方式
type Option func(o *healthCheckOptions)

// 健康检查客户端4个关注的参数
type healthCheckOptions struct {
  initialDelay      time.Duration
  requestTimeout    time.Duration
  failureThreshold  int
  interval          time.Duration
  successStatusCode int
}

// StartEndpointHealthCheck 客户端对客户端的健康检查参数赋值后，再启动周期性探测主服务的健康状态
//
// 并把主服务的健康状态通过返回值singalChan channel来实时传递给主服务
func StartEndpointHealthCheck(endpointAddress string, opts ...Option) chan bool {
  // 初始化健康检查客户端参数和赋值
  options := &healthCheckOptions{}
  applyDefaults(options)

  for _, o := range opts {
      o(options)
  }

  // 设置实时返回给主服务的健康状态，true表示健康，false表示不健康
  signalChan := make(chan bool, 1)

  // 启动健康检查客户端goroutine，并周期性对主服务健康进行探测
  go func(ch chan<- bool, endpointAddress string, options *healthCheckOptions) {
      // 初始化等待健康检查端口服务启动完成
      ticker := time.NewTicker(options.interval)
      failureCount := 0
      time.Sleep(options.initialDelay)

      // 创建http client并周期性的发起健康探测, 路由: /healthz
      client := &fasthttp.Client{
          MaxConnsPerHost:           5, // Limit Keep-Alive connections
          ReadTimeout:               options.requestTimeout,
          MaxIdemponentCallAttempts: 1,
      }

      req := fasthttp.AcquireRequest()
      req.SetRequestURI(endpointAddress)
      req.Header.SetMethod(fasthttp.MethodGet)
      defer fasthttp.ReleaseRequest(req)

      // 周期性地执行健康探测任务
      for range ticker.C {
          resp := fasthttp.AcquireResponse()
          // 发起http client请求
          err := client.DoTimeout(req, resp, options.requestTimeout)
          // 如果请求报错，或者返回的http状态码不是自定义的状态码，则进入连续错误统计
          if err != nil || resp.StatusCode() != options.successStatusCode {
              failureCount++
              // 当连续探测失败次数，达到了设置的失败阈值，则返回给主服务当前状态不健康
              if failureCount == options.failureThreshold {
                  ch <- false
                }
            } else {
                // 否则表示状态健康，且重置failureCount为0，失败必须是连续的，才会发生统计
                ch <- true
                failureCount = 0
            }
            fasthttp.ReleaseResponse(resp)
        }
    }(signalChan, endpointAddress, options)
    // 返回signalChan给主调方，来告知当前服务的健康状态
    return signalChan
}
```

## 健康检查参数

目前参数也就是针对前面说的健康检查服务客户端的5个参数设置, 分别是
1. FailureThreshold， 连续失败阈值；
2. InitialDelay, 等待健康检查服务就绪
3. Interval， 健康检查探测周期
4. RequestTimeout，http请求健康检查服务的超时时间
5. SuccessStatusCode，设置健康检查服务响应成功的响应状态码

## 总结

通过这个heath健康检查包，就可以启动健康检查服务框架，以及使用健康检查客户端，开箱即用，然后再和自己主服务结合，来实现服务的健康检查。同时注意，使用了signals.Context包上下文，则可以在服务的整个链路和所有协程中，都用该上下文来监听主服务的退出信号做一些收尾处理
