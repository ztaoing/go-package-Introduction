# gokit-zipkin
# 开发和测试设置
为了使Zipkin更易于测试，开发和试验，已经做出了巨大的努力。 Zipkin现在可以从单个Docker容器运行，也可以运行其自包含的可执行jar，而无需进行大量配置。 在其默认配置中，您将在端口9411上使用HTTP收集器，内存跨度存储后端和Web UI运行Zipkin。

举例：
`docker run -d -p 9411:9411 openzipkin/zipkin`

> 中间件的使用

按照addsvc示例查看如何连接Zipkin中间件。 所做的更改应该相对较小。

zipkin-go程序包使Reporters可以将Span发送到Zipkin HTTP和Kafka Collector

> 配置Zipkin HTTP Reporter

要将HTTP Reporter与在localhost上运行的Zipkin实例一起使用，您需要引导zipkin-go，如下所示：

```
var (
  serviceName        = "MyService"
  serviceHostPort    = "localhost:8000"
  zipkinHTTPEndpoint = "http://localhost:9411/api/v2/spans"
)

// create an instance of the HTTP Reporter.
reporter := zipkin.NewReporter(zipkinHTTPEndpoint)

// create our tracer's local endpoint (how the service is identified in Zipkin).
localEndpoint, err := zipkin.NewEndpoint(serviceName, serviceHostPort)

// create our tracer instance.
tracer, err = zipkin.NewTracer(reporter, zipkin.WithLocalEndpoint(localEndpoint))
```
> 追踪资源

这是如何跟踪资源并使用本地范围的示例。

```
import (
	zipkin "github.com/openzipkin/zipkin-go"
)

func (svc *Service) GetMeSomeExamples(ctx context.Context, ...) ([]Examples, error) {
  // Example of annotating a database query:
  var (
    spanContext model.SpanContext
    serviceName = "MySQL"
    serviceHost = "mysql.example.com:3306"
    queryLabel  = "GetExamplesByParam"
    query       = "select * from example where param = :value"
  )

  // retrieve the parent span from context to use as parent if available.
  if parentSpan := zipkin.SpanFromContext(ctx); parentSpan != nil {
    spanContext = parentSpan.Context()
  }

  // create the remote Zipkin endpoint
  ep, _ := zipkin.NewEndpoint(serviceName, serviceHost)

  // create a new span to record the resource interaction
  span := zipkin.StartSpan(
    queryLabel,
    zipkin.Parent(parentSpan.Context()),
    zipkin.WithRemoteEndpoint(ep),
  )

	// add interesting key/value pair to our span
	span.SetTag("query", query)

	// add interesting timed event to our span
	span.Annotate(time.Now(), "query:start")

	// do the actual query...

	// let's annotate the end...
	span.Annotate(time.Now(), "query:end")

	// we're done with this span.
	span.Finish()

	// do other stuff
	...
}
```