本文主要分析三个dapr包，一个为concurrency, 另一个为fswatcher，最后一个为logger

1. concurrency用于dapr中相关的批量操作替代操作。比如：如果某个第三方软件服务提供商，不支持批量操作，则可以通过concurrency包，来执行并发批量操作。
2. fswatcher用于监听文件系统的订阅文件目录或者文件路径的变更，并告诉订阅者事件和相应地处理。比如：证书文件的监听和重新创建, 过期文件删除和重新创建新的证书，使得证书永远保证有效
3. logger作为dapr的通用模块，到处都在使用，保障日志文件的落地，包括文件的大小、日期和保留周期等策略

## concurrency

这个concurrency包，是对[limiter](https://github.com/korovkin/limiter)的简化，它最主要的作用，是对并发进行控制，比如：只允许10个请求并发，则100请求的到来，则会保证运行的goroutine数据最多等于10个，超过并发数，则等待运行队列上的其他任务执行完成后，再执行阻塞的其他等待请求任务。

下面的代码，包含三个方法：
1. NewLimiter，初始化创建一个limiter，设置请求并发数
2. Execute，把请求放入到阻塞队列或者运行队列中，等待请求任务被执行。
3. Wait，等待阻塞队列和运行队列上的所有请求任务，都处于空闲状态.

```golang
// Limiter 并发控制类
type Limiter struct {
  // 最大任务执行并发量
	limit         int
  // 运行队列，长度为limit
	tickets       chan int
  // 当前运行队列中，正在运行的任务数量
	numInProgress int32
}

// NewLimiter创建一个Limit并发控制实例对象
func NewLimiter(limit int) *Limiter {
  // 如果limit输入参数不合法，直接给个默认并发任务最大值100
	if limit <= 0 {
		limit = DefaultLimit
	}

	// 初始化Limiter实例，设置运行任务队列的最大长度
	c := &Limiter{
		limit:   limit,
		tickets: make(chan int, limit),
	}

	// 首先生产10个票据，放入队列中，使得队列变成可运行的任务队列
	for i := 0; i < c.limit; i++ {
		c.tickets <- i
	}

	return c
}

// Execute 把请求任务放在阻塞队列或者运行队列中，等待被执行
func (c *Limiter) Execute(job func(param interface{}), param interface{}) int {
	ticket := <-c.tickets
	atomic.AddInt32(&c.numInProgress, 1)
	go func(param interface{}) {
		defer func() {
			c.tickets <- ticket
			atomic.AddInt32(&c.numInProgress, -1)
		}()

		// run the job
		job(param)
	}(param)
	return ticket
}

// Wait 等待阻塞队列和运行队列的任务数量为空，则表示一次并发的所有请求通过多批次的运行，最终执行完成了。
func (c *Limiter) Wait() {
	for i := 0; i < c.limit; i++ {
		<-c.tickets
	}
}
```

通过这个简单的concurrency包，在grpc和http中，在处理GetBulkState中，获取批量存储数据时，如果使用到的第三方软件存储服务提供商，不支持批量操作，则只能采用降级地方式，通过concurrency包，并发执行任务，来批量操作。这个操作并不是事务型。

这里比较有意思的一个点：[golang 下面有无下划线占位符的区别是什么？](https://mk.woa.com/q/269412/answer/71677)

我发现 `_ = <-c.tickets`, 在静态检查时不能通过，报下面的错误: S1005

如果改为`<-c.tickets`，则静态检查不会报错，但是我们的CodeCC会报需要修复吗？

```errors
Running [/home/runner/golangci-lint-1.31.0-linux-amd64/golangci-lint run --out-format=github-actions] in [] ...
 Error: S1023: redundant `return` statement (gosimple)
 Error: S1005: unnecessary assignment to the blank identifier (gosimple)
 Error: S1005: unnecessary assignment to the blank identifier (gosimple)

 Error: issues found
 Ran golangci-lint in 19135ms
```

## fswatcher
fswatcher包实现对文件系统文件或者目录的监控，是通过[fsnotify](github.com/fsnotify/fsnotify)第三方库实现的, 当`dir`下有文件变更，包括创建、删除、改变文件属性、写入等等操作，都会发生事件订阅通知


```golang
func Watch(ctx context.Context, dir string, eventCh chan<- struct{}) error {
		// 创建一个watcher监听对象
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        return errors.Wrap(err, "failed to create watcher")
    }
    defer watcher.Close()

		// 把要监听的目录或者文件，订阅给watcher。当有事件产生时会通过channel管道传出。并给发送给订阅的消费者。
    if err := watcher.Add(dir); err != nil {
        return errors.Wrap(err, "watcher error")
    }

LOOP:
    for {
        select {
        // watch for events
        case event := <-watcher.Events:
						// 这里订阅了dir下的文件创建和文件写入订阅消息
            if event.Op == fsnotify.Create || event.Op == fsnotify.Write {
								// 校验是否为自己订阅的dir
                if strings.Contains(event.Name, dir) {
                    // 当发生订阅事件通知后，等待1s，保证文件完全有时间更新完成。
										// 以保证可以读取到完整的数据，该库目前主要是给dapr sentry服务使用。
                    time.Sleep(time.Second * 1)
                    eventCh <- struct{}{}
                }
            }
        case <-watcher.Errors:
            break LOOP
        case <-ctx.Done():
            break LOOP
        }
    }
    return nil
}
```

比如，在dapr/sentry服务创建过程中，会指定CA根证书和Issuer证书颁布者的证书和密钥路径，然后再监听该目录，如果该目录有文件创建或者写入，则会重启该dapr sentry服务。

下面，我们看看dapr sentry服务在启动过程中，有关fswatcher监听包的操作和使用

```golang
cmd/sentry/main.go

// 启动dapr sentry服务
func main(){
	// credsPath 表示CA机构和Issuer证书颁布者的证书和私钥文件信息，如果文件不存在则会使用CA机构进行自签署和签署
	configName := flag.String("config", defaultDaprSystemConfigName, "Path to config file, or name of a configuration object")
	// credsPath 默认证书路径：/var/run/dapr/credentials
  credsPath := flag.String("issuer-credentials", defaultCredentialsPath, "Path to the credentials directory holding the issuer data")
	// trustDomain 表示信任域，通过SPIFFE标准保护内部服务RPC不受恶意攻击
  trustDomain := flag.String("trust-domain", "localhost", "The CA trust domain")
  flag.Parse()
	......

	// 证书目录下三个文件，分别是CA机构公钥证书: ca.crt
	// Issuer证书颁布者 公钥证书：issuer.crt 和私钥证书: issuer.key
  issuerCertPath := filepath.Join(*credsPath, credentials.IssuerCertFilename)
  issuerKeyPath := filepath.Join(*credsPath, credentials.IssuerKeyFilename)
  rootCertPath := filepath.Join(*credsPath, credentials.RootCertFilename)

	// 监听程序挂掉的信息列表，当捕获到订阅的信号时，再做后续dapr sentry服务的优雅退出处理
  stop := make(chan os.Signal, 1)
  signal.Notify(stop, os.Interrupt, syscall.SIGTERM)

  ctx := signals.Context()
  config, err := config.FromConfigName(*configName)
  if err != nil {
      log.Warn(err)
  }
  config.IssuerCertPath = issuerCertPath
  config.IssuerKeyPath = issuerKeyPath
  config.RootCertPath = rootCertPath
  config.TrustDomain = *trustDomain

	// 如果是默认配置，则监听/var/run/dapr/credentials目录, 主要就是对CA机构证书、Issuer证书颁布者的公钥证书和私钥证书进行监听
  watchDir := filepath.Dir(config.IssuerCertPath)
	// 创建dapr sentry服务
  ca := sentry.NewSentryCA()

  log.Infof("starting watch on filesystem directory: %s", watchDir)
  issuerEvent := make(chan struct{})
  ready := make(chan bool)

	// 启动dapr sentry服务，为其他grpc服务颁发和签署数字证书
  go ca.Run(ctx, config, ready)

	// 服务创建成功后，通过ready表示服务启动成功，之前处于阻塞状态
  <-ready

	// 服务启动成功后，开始对CA机构证书和Issuer证书和私钥文件进行监听
	//
	// 并通过issuerEvent反馈事件， 只要有订阅的事件发生，就重启dapr sentry服务重新加载证书或者创建证书
  go fswatcher.Watch(ctx, watchDir, issuerEvent)

  go func() {
			// 如果有监听的目录，有证书创建和写入，则重启dapr sentry服务, 这样可以保证永久有效
      for range issuerEvent {
          monitoring.IssuerCertChanged()
          log.Warn("issuer credentials changed. reloading")
          ca.Restart(ctx, config)
			}
	}
	......
	<-stop
	......
}
```

## logger

dapr logger的默认实现方式只有一种，且目前没有对外提供注册和实现。并且采用[logrus](github.com/sirupsen/logrus), 但是目前市面上有更优秀的[zaplog](github.com/uber-go/zap)，后面应该是会插件化实现log注册的。

目前dapr logger的默认实现方式，有以下几个特点：
1. 日志输出只能是终端的方式，不支持文件落地；
2. 日志输出格式，支持json或者文本输出；
3. 可以动态调整日志输出级别。
4. 日志支持的级别，包括：debug, info, warn, error, fatal
5. 同时增加了日志的类型(默认为log，可以设置为request请求类型日志等)，设置日志app服务名称(如：dapr)， 主机名，dapr版本等参数。

这里只简单地介绍下如何使用, 下面这个例子，首先设置日志相关参数Options，并调整日志输出级别。注意，当设置日志参数后，需要应用到所有的日志服务实例中，方法：**logger.ApplyOptionsToLoggers**

```golang
package main

import "github.com/dapr/dapr/pkg/logger"

func main() {
    option := logger.DefaultOptions()
    option.SetOutputLevel("error")
    option.JSONFormatEnabled = true
    instance := logger.NewLogger("dapr")
    logger.ApplyOptionsToLoggers(&option)
    instance.Info("dapr-info ", "dapr hello,world")
    instance.Warn("dapr-warn ", "dapr hello,world")
    instance.Error("dapr-error ", "dapr hello,world")
}
```

目前在dapr中，日志的初始化主要在下面的代码目录中：

```golang
./cmd/sentry/main.go:43:        loggerOptions := logger.DefaultOptions()
./cmd/operator/main.go:50:      loggerOptions := logger.DefaultOptions()
./cmd/injector/main.go:65:      loggerOptions := logger.DefaultOptions()
./cmd/placement/config.go:70:   cfg.loggerOptions = logger.DefaultOptions()
./pkg/runtime/cli.go:50:        loggerOptions := logger.DefaultOptions()
```

前四个结果分别是，dapr-system命名空间下的dapr-sentry, dapr-operator, dapr-sidecar-injector和dapr-placement四个POD系统服务。

最后一个是业务POD中的daprd runtime sidecar容器服务。

所以在dapr产品中，所有的服务都是使用了dapr/pkg/logger包下的日志组件，来实现自己的日志输出能力。

## 总结

通过前面介绍的concurrency、fswatcher和logger，分别实现：
1. dapr http/grpc的GetBulkState批量操作能力(当第三方软件服务提供商不支持批量操作时，则通过单个批量执行达到)；而concurrency用来控制并发能力
2. fswatcher借助第三方库，来实现对CA根证书、Issuer证书颁布者的证书和私钥进行文件监控。当证书发生变化时，重启dapr sentry服务，并更新CA根证书和Issuer证书颁布者和的证书和私钥；
3. logger组件用于为dapr-system命名空间下的四个系统服务提供日志能力，并且为业务pod中的daprd runtime sidecar容器服务提供日志能力

**留下一个问题**： 当dapr sentry系统服务重启后，则原来这个服务已经为grpc外部服务颁布了数字签名，这些签名失效了吗？怎么办？
