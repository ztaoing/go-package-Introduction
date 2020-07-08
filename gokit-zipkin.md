# gokit-zipkin
# 开发和测试设置
为了使Zipkin更易于测试，开发和试验，已经做出了巨大的努力。 Zipkin现在可以从单个Docker容器运行，也可以运行其自包含的可执行jar，而无需进行大量配置。 在其默认配置中，您将在端口9411上使用HTTP收集器，内存跨度存储后端和Web UI运行Zipkin。

举例：
`docker run -d -p 9411:9411 openzipkin/zipkin`

> 中间件的使用

按照[addsvc](https://github.com/go-kit/kit/tree/master/examples/addsvc)示例可以知道如何连接Zipkin中间件。 所做的更改应该相对较小。

zipkin-go程序包使Reporters可以将Span发送到Zipkin HTTP和Kafka Collector

> 配置Zipkin HTTP Reporter

要将HTTP Reporter与在localhost上运行的Zipkin实例一起使用，您需要引导zipkin-go，如下所示：

```
var (
  serviceName        = "MyService"
  serviceHostPort    = "localhost:8000"
  zipkinHTTPEndpoint = "http://localhost:9411/api/v2/spans"
)

// 创建 HTTP Reporter实例.
reporter := zipkin.NewReporter(zipkinHTTPEndpoint)

// 创建链路追踪的本地的 endpoint (how the service is identified in Zipkin).
localEndpoint, err := zipkin.NewEndpoint(serviceName, serviceHostPort)

// 创建tracer实例
tracer, err = zipkin.NewTracer(reporter, zipkin.WithLocalEndpoint(localEndpoint))
```
> 追踪资源

这是如何跟踪资源并使用本地范围的示例。

```
import (
	zipkin "github.com/openzipkin/zipkin-go"
)

func (svc *Service) GetMeSomeExamples(ctx context.Context, ...) ([]Examples, error) {
  // 注释数据库查询的示例:
  var (
    spanContext model.SpanContext
    serviceName = "MySQL"
    serviceHost = "mysql.example.com:3306"
    queryLabel  = "GetExamplesByParam"
    query       = "select * from example where param = :value"
  )

  // 从context找回 parent span 
    if parentSpan := zipkin.SpanFromContext(ctx); parentSpan != nil {
    spanContext = parentSpan.Context()
  }

  //创建远端的Zipkin endpoint
  ep, _ := zipkin.NewEndpoint(serviceName, serviceHost)

  // 创建一个新的span以记录资源交互 
   span := zipkin.StartSpan(
    queryLabel,
    zipkin.Parent(parentSpan.Context()),
    zipkin.WithRemoteEndpoint(ep),
  )

	// 在span中增加 key/value 键值对
		span.SetTag("query", query)

	// 在span中增加定时时间
		span.Annotate(time.Now(), "query:start")

	// 执行实际的查询操作...

	// 增加结束事件...
	span.Annotate(time.Now(), "query:end")

	// span的处理已经完成.
	span.Finish()

	// 做其他的事情
	...
}
```