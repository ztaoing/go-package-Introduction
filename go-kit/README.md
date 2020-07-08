#go-kit
-------

`* auth/basic	` 
`* auth/casbin	`
`* auth/jwt	`

-------

`* circuitbreaker`	断路器实现了断路器模式。

-------

`* cmd/kitgen	`
`* cmd/kitgen/templates`

-------

`endpoint	`: [端点定义了RPC的抽象](https://github.com/ztaoing/go-package-Introduction/blob/master/go-kit/1%E3%80%81endpoint/endpoint.md)

-------

**examples**
* examples/addsvc/cmd/addcli	
* examples/addsvc/cmd/addsvc	
* examples/addsvc/pb	
* examples/addsvc/pkg/addendpoint	
* examples/addsvc/pkg/addservice	
* examples/addsvc/pkg/addtransport	
* examples/addsvc/thrift/gen-go/addsvc	
* examples/addsvc/thrift/gen-go/addsvc/add_service-remote	
* examples/apigateway	
* examples/profilesvc	

`* examples/profilesvc/client` 客户端根据预定义的Consul服务名称和相关标签提供一个profilevc客户端。

`* examples/shipping	`

`* examples/shipping/booking`预订提供了预订货物的用例。

`* examples/shipping/cargo`	域模型的核心。

`* examples/shipping/handling`提供注册事件的用例。

`* examples/shipping/inmem`	提供所有域存储库的内存实现。
`* examples/shipping/inspection`提供了检查的方式

`* examples/shipping/location` location provides the Location aggregate.

`* examples/shipping/routing`	Package routing provides the routing domain service.
`* examples/shipping/tracking`	Package tracking provides the use-case of tracking a cargo.
`* examples/shipping/voyage`	Package voyage provides the Voyage aggregate.

* examples/stringsvc1	

* examples/stringsvc2	

* examples/stringsvc3	

* examples/stringsvc4	

-------

**日志包，提供了结构化的logger**
`* log/deprecated_levels	`
`* log/level`	Package level implements leveled logging on top of Go kit's log package.
`* log/logrus`	Package logrus provides an adapter to the go-kit log.Logger interface.
`* log/syslog`	
`* log/term`	Package term provides tools for logging to a terminal.
`* log/zap	`

-------

**metrics	为应用程序检测提供了框架**
* metrics/cloudwatch	
* metrics/cloudwatch2	Package cloudwatch2 emits all data as a StatisticsSet (rather than a singular Value) to CloudWatch via the aws-sdk-go-v2 SDK.
* metrics/discard	Package discard provides a no-op metrics backend.
* metrics/dogstatsd	Package dogstatsd provides a DogStatsD backend for package metrics.
* metrics/expvar	Package expvar provides expvar backends for metrics.
* metrics/generic	Package generic implements generic versions of each of the metric types.
* metrics/graphite	Package graphite provides a Graphite backend for metrics.
* metrics/influx	Package influx provides an InfluxDB implementation for metrics.
* metrics/influxstatsd	Package influxstatsd provides support for InfluxData's StatsD Telegraf plugin.
* metrics/internal/convert	Package convert provides a way to use Counters, Histograms, or Gauges as one of the other types
* metrics/internal/lv	
* metrics/internal/ratemap	Package ratemap implements a goroutine-safe map of string to float64.
* metrics/multi	Package multi provides adapters that send observations to multiple metrics simultaneously.
* metrics/pcp	
* metrics/prometheus	Package prometheus provides Prometheus implementations for metrics.
* metrics/provider	Package provider provides a factory-like abstraction for metrics backends.
* metrics/statsd	Package statsd provides a StatsD backend for package metrics.
* metrics/teststat	Package teststat provides helpers for testing metrics backends.

-------

**ratelimit**
* ratelimit	

-------

**sd提供与服务发现相关的实用程序**
* sd/consul	Package consul provides Instancer and Registrar implementations for Consul.
* sd/dnssrv	Package dnssrv provides an Instancer implementation for DNS SRV records.
* sd/etcd	Package etcd provides an Instancer and Registrar implementation for etcd.
* sd/etcdv3	Package etcdv3 provides an Instancer and Registrar implementation for etcd v3.
* sd/eureka	Package eureka provides Instancer and Registrar implementations for Netflix OSS's Eureka
* sd/internal/instance	
* sd/lb	Package lb implements the client-side load balancer pattern.
* sd/zk	Package zk provides Instancer and Registrar implementations for ZooKeeper.

-------

**tracing	提供用于分布式跟踪的帮助程序和绑定**
 
* tracing/opencensus	Package opencensus provides Go kit integration to the OpenCensus project.
* tracing/opentracing	Package opentracing provides Go kit integration to the OpenTracing project.
* tracing/zipkin	Package zipkin provides Go kit integration to the OpenZipkin project through the use of zipkin-go, the official OpenZipkin tracer implementation for Go.

-------

**transport	包含适用于所有受支持transport的帮手**
* transport/amqp	Package amqp implements an AMQP transport.
* transport/awslambda	Package awslambda provides an AWS Lambda transport layer.

`transport/grpc`	提供了一个gRPC 对endpoints绑定.

* transport/grpc/_grpc_test	
* transport/grpc/_grpc_test/pb	

` transport/http` 对endpoints生成一个目标 HTTP 

* transport/http/jsonrpc	Package jsonrpc provides a JSON RPC (v2.0) binding for endpoints.
* transport/http/proto	
* transport/httprp	Package httprp provides an HTTP reverse-proxy transport.
* transport/nats	Package nats provides a NATS transport.
-------

**conn提供与连接有关的实用程序。**
* util/conn


-------
# go-kit介绍 （以下非本人翻译，转自 [简书 李小贱AA](https://www.jianshu.com/p/31108ba61aee)）
go kit 是一个分布式的开发工具集，在大型的组织(业务)中可以用来构建微服务。其解决了分布式系统中常见的问题，因此，使用者可以将精力集中在业务逻辑上。

##Go-kit 组件介绍

**2.1 Endpoint**

Endpoint：Go-kit 首选解决了RPC消息模式。其中使用了一个抽象的endpoint来为每一个RPC建立模型。

endpoint通过被一个server进行实现(implement),或是被一个client调用。这是很多Go-kit 组件的基本构建代码块

**2.2 Circuit breaker(回路断路器)**

回路断路器:模块提供额许多流行的回路断路lib的端点(endpoint)适配器。回路断路器可以避免雪崩，并且提高了针对间歇性错误的弹性.每个client的端点都应该封装(wrapped)在回路断路器中。

**2.3 Rate limiter(限流器)**

限流器：模块提供了到限流器代码包的端点适配器。限流器对服务端(server-client) 和 客户端(client-side) 同等生效。使用限流器可以强制进，出 请求量在阈值上限以下。

**2.4 Transport(传输层)**

传输层：提供了将特定的序列化算法绑定到端点的辅助方法。当前，Go kit  只针对JSON和HTTP提供了辅助方法。如果你的组织使用了完整功能的传输层，典型的方案是使用GO在传输层提供的函数。 Go-kit 并不需要来做太多的事情。这些情况，可以查阅代码例子来理解如何为你的端点写一个适配器。

**2.5 Logging(日志)**

服务产生的日志是会被延迟消费（使用）的，或者是人或者是机器（来使用）。人可能会对调试错误、跟踪特殊的请求感兴趣。机器可能会对统计那些有趣的事件，或是对离线处理的结果进行聚合。这两种情况，日志消息的结构化和可操作性是很重要的。Go kit的log模块针对这些实践提供了最好的设计。

**2.6 Metrics/Instrumentation 度量/仪表盘**

直到服务经过了跟踪计数、延迟、健康状况和其他的周期性的或针对每个请求信息的仪表盘化，才能被认为是“生产环境”完备的。Go kit 的metric模块为你的服务提供了通用并健壮的接口集合。可以绑定到常用的后端服务，比如expvar、statsd、Prometheus。

**2.7 Request tracing（请求跟踪）**

随着你的基础设施的增长，能够跟踪一个请求变得越来越重要，因为它可以在多个服务中进行穿梭并回到用户。Go kit的tracing模块提供了为端点和传输的增强性的绑定功能，以捕捉关于请求的信息，并把它们发送到跟踪系统中。（当前支持Zipkin，计划支持Appdash

**2.8 Service discovery and load balancing（服务发现和负载均衡）**

如果你的服务调用了其他的服务，需要知道如何找到它（另一个服务），并且应该智能的将负载在这些发现的实例上铺开（即，让被发现的实例智能的分担服务压力）。Go kit的loadbalancer模块提供了客户端端点的中间件来解决这类问题，无论你是使用的静态的主机名还是IP地址，或是 DNS的SRV记录，Consul，etcd 或是 Zookeeper。并且，如果你使用定制的系统，也可以非常容易的编写你自己的Publisher，以使用 Go kit 提供的负载均衡策略。（目前，支持静态主机名、etcd、Consul、Zookeeper）
