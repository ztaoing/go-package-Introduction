# OpenTracing API for Go 分布式链路追踪规范
OpenTracing是一个分布式链路追踪的管理平台，可以在平台的基础上实现插拔式的链路追踪组件的融合
## Required Reading
为了理解Go平台的API，首先必须更熟悉OpenTracing项目和术语
[https://opentracing.io/guides/golang/quick-start/](https://opentracing.io/guides/golang/quick-start/)
## API overview for those adding instrumentation
这个opentracing包的使用者实际上只需要关注几个关键的抽象：StartSpan函数，Span接口以及在main（）时绑定一个Tracer。 以下是代码片段，展示了一些重要的用例。

* 单例初始化
最简单的开始时从 ./default_tracer.go. 非常简单而且可行

```
import "github.com/opentracing/opentracing-go"
    import ".../some_tracing_impl"

    func main() {
        opentracing.SetGlobalTracer(
            // 链路追踪的具体实现:
            some_tracing_impl.New(...),
        )
        ...
    }
```
* 非单例初始化
If you prefer direct control to singletons, manage ownership of the opentracing.Tracer implementation explicitly.
如果您更喜欢直接控制单例，则显式管理opentracing.Tracer。
* 为给定的 Go context.Context创建一个span
如果在应用程序中使用context.Context，OpenTracing的Go库将很容易依靠它进行Span传播。 要启动新的（阻塞的子级的span）Span，可以使用StartSpanFromContext。

```
func xyz(ctx context.Context, ...) {
        ...
        span, ctx := opentracing.StartSpanFromContext(ctx, "operation_name")
        defer span.Finish()
        span.LogFields(
            log.String("event", "soft error"),
            log.String("type", "cache timeout"),
            log.Int("waited.millis", 1500))
        ...
    }
```
**通过创建一个"root span"来启动一个空的trace**
总是可以创建没有父级或其他因果引用的“root”span

```
 func xyz() {
        ...
        sp := opentracing.StartSpan("operation_name")
        defer sp.Finish()
        ...
    }
```
**为(父) Span创建一个(子) Span**

```
  func xyz(parentSpan opentracing.Span, ...) {
        ...
        sp := opentracing.StartSpan(
            "operation_name",
            //指定父span
        opentracing.ChildOf(parentSpan.Context()))
        defer sp.Finish()
        ...
    }
```
**序列化到wire**

```
func makeSomeRequest(ctx context.Context) ... {
        if span := opentracing.SpanFromContext(ctx); span != nil {
            httpClient := &http.Client{}
            httpReq, _ := http.NewRequest("GET", "http://myservice/", nil)

            // 把发送span的TraceContext作为 HTTP的request的头信息
            opentracing.GlobalTracer().Inject(
                span.Context(),
                opentracing.HTTPHeaders,
                opentracing.HTTPHeadersCarrier(httpReq.Header))

            resp, err := httpClient.Do(httpReq)
            ...
        }
        ...
    }
```
**从wire反序列化**

```
http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
        var serverSpan opentracing.Span
        appSpecificOperationName := ...
        wireContext, err := opentracing.GlobalTracer().Extract(
            opentracing.HTTPHeaders,
            opentracing.HTTPHeadersCarrier(req.Header))
        if err != nil {
            // Optionally record something about err here
        }

        // 如有有可用的rpc client 可以创建span指向rpc client
        // 如果 wireContext == nil, 会创建一个 root span 
        serverSpan = opentracing.StartSpan(
            appSpecificOperationName,
            ext.RPCServerOption(wireContext))

        defer serverSpan.Finish()

        ctx := opentracing.ContextWithSpan(context.Background(), serverSpan)
        ...
    }
```
**使用log.Noop有条件地捕获字段**
在某些情况下，您可能希望动态决定是否记录一个字段。 例如，您可能想在非生产环境中捕获其他数据，例如客户ID：

```
func Customer(order *Order) log.Field {
        //测试环境
        if os.Getenv("ENVIRONMENT") == "dev" {
            return log.String("customer", order.Customer.ID)
        }
        return log.Noop()
    }
```

# Quick Start
本页上的示例使用Jaeger（与OpenTracing兼容的跟踪器）。 这些示例假定Jaeger多合一映像通过Docker在本地运行：

```
$ docker run -d -p 6831:6831/udp -p 16686:16686 jaegertracing/all-in-one:latest
```
通过调整初始化代码以使其与特定实现相匹配，可以轻松地将它们调整为使用其他OpenTracing兼容的Tracer。
**设置追踪器**

```
import (
    "log"
    "os"

    opentracing "github.com/opentracing/opentracing-go"
    "github.com/uber/jaeger-lib/metrics"

    "github.com/uber/jaeger-client-go"
    jaegercfg "github.com/uber/jaeger-client-go/config"
    jaegerlog "github.com/uber/jaeger-client-go/log"
)

...

func main() {
    // Sample configuration for testing. Use constant sampling to sample every trace
    // and enable LogSpan to log every span via configured Logger.
    //用于测试的样本配置。 使用恒定采样来采样每个跟踪，并使LogSpan能够通过配置的Logger记录每个span。
    cfg := jaegercfg.Configuration{
        ServiceName: "your_service_name",
        Sampler:     &jaegercfg.SamplerConfig{
            Type:  jaeger.SamplerTypeConst,
            Param: 1,
        },
        Reporter:    &jaegercfg.ReporterConfig{
            LogSpans: true,
        },
    }

    // 记录日志和指标信息使用 github.com/uber/jaeger-client-go/log
    // 和 github.com/uber/jaeger-lib/metrics 分别绑定到真正的  logging 和 metrics 框架
       jLogger := jaegerlog.StdLogger
    jMetricsFactory := metrics.NullFactory

    // 使用日志和指标组件初始化一个链路追踪
    tracer, closer, err := cfg.NewTracer(
        jaegercfg.Logger(jLogger),
        jaegercfg.Metrics(jMetricsFactory),
    )
    // 使用 Jaeger tracer组件来设置单例的opentracing.Tracer.
    opentracing.SetGlobalTracer(tracer)
    defer closer.Close()

    // continue main()
}

```
**启动一个链路追踪器**

```
import (
    opentracing "github.com/opentracing/opentracing-go"
)

...

tracer := opentracing.GlobalTracer()

span := tracer.StartSpan("say-hello")
println(helloStr)
span.Finish()
```
**创建一个子级span**

```
import (
    opentracing "github.com/opentracing/opentracing-go"
)

...

tracer := opentracing.GlobalTracer()

parentSpan := tracer.StartSpan("parent")
defer parentSpan.Finish()

...

// 使用ChildOf选项，创建一个子级span.
childSpan := tracer.StartSpan(
    "child",
    opentracing.ChildOf(parentSpan.Context()),
)
defer childSpan.Finish()
```
**创建一个 HTTP 请求**

我们通过将Context上下文注入http headers中以达到在服务之间传播目的。 一旦，下游服务收到http请求后，必须提取上下文并继续跟踪。 
注意：（代码示例无法正确处理错误，请不要在生产代码中执行此操作；这只是一个示例）

**上游服务(client)**

```
import (
    "net/http"

    opentracing "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

...

tracer := opentracing.GlobalTracer()

clientSpan := tracer.StartSpan("client")
defer clientSpan.Finish()

url := "http://localhost:8082/publish"
req, _ := http.NewRequest("GET", url, nil)

// 通过在clientSpan上设置一些tags，来标明它是client 端的span  ，额外的 HTTP tags 对调试很有帮助.
ext.SpanKindRPCClient.Set(clientSpan)
ext.HTTPUrl.Set(clientSpan, url)
ext.HTTPMethod.Set(clientSpan, "GET")

// 将client端span的上下文注入到headers中
tracer.Inject(clientSpan.Context(), opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(req.Header))
resp, _ := http.DefaultClient.Do(req)
```
**下游服务(server)**

```
import (
    "log"
    "net/http"

    opentracing "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

func main() {

    // 链路追踪器初始化, etc.

    ...

    http.HandleFunc("/publish", func(w http.ResponseWriter, r *http.Request) {
        // 从headers中提取上下文context
        spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))
        serverSpan := tracer.StartSpan("server", ext.RPCServerOption(spanCtx))
        defer serverSpan.Finish()
    })

    log.Fatal(http.ListenAndServe(":8082", nil))
}
```
**展示链路追踪**
如果你有 Jaeger 一起运行, 你可以在localhost:16686查看链路追踪的效果.

**Link to GO walkthroughs / tutorials**
* Take OpenTracing for a HotROD Ride involves successive optimizations of a Go-based Ride-on-Demand demonstration service, all informed by tracing data.
* In-depth Self-Guided Golang Opentracing Tutorial

# Spans
此页面正在建设中。
OpenTracing Go API允许线程中的一个跨度在任何时间点都处于活动状态。 同一线程可能涉及其他范围，它们满足以下条件：
* Started,
* Not finished,
* Not “active”.
如果跨度为：同一线程上可以有多个跨度：
* Waiting for I/O,
* Blocked on a child Span or
* Off of the critical path
**访问当前激活状态的span**
开发人员可以通过范围访问任何活动范围，如下所示：

```
 ** Go code snippet here **
```
**在线程之间传递span**

使用OpenTracing API，开发人员可以在不同线程之间传输跨度。 跨度的生命周期可能在一个线程中开始，而在另一个线程中结束。 不支持将范围传递给另一个线程或回调。

**span：约定和标准**

**操作名称和基数**
应用程序和库开发人员需要指定每个范围的操作名称。 操作名称是跨度类的通用名称，代表唯一的实例。 以下是在Go中使用操作名称“ say-hello”初始化跨度的语句：

```
** Go code snippet here **
```

选择通用操作名称的原因是允许跟踪系统进行聚合。 例如，Jaeger跟踪器可以选择针对通过应用程序的所有流量发出指标。 每个跨度都有唯一的操作名称将使指标失去作用。 对于希望在跟踪中捕获程序参数以区分它们的应用程序或库开发人员，建议的解决方案是用标签或日志注释跨区。 这些将在下面的部分中讨论。

每个操作名称都应具有一定的语义含义，并且应该对其进行选择，以使其基数既不太高也不太低。 操作名称的基数定义如下：

```
card(operation X) = total number of spans of operation X
```
例如，当使用http.url（作为操作名称）时，基数会太高，另一方面，如果选择http.method作为操作名称，则基数会太低。

**标准标签**
标签是一个键值对，它提供有关开发人员作为工具的一部分定义的跨度的某些元数据。 标签描述了适用于整个跨度持续时间的跨度属性。 例如，如果一个跨度代表一个HTTP请求，则该请求的URL应该记录为一个标记，因为它在跨度的整个生命周期中保持不变。
考虑为将“ Hello Bryan”打印到控制台而编写的程序。 在这种情况下，字符串“ Bryan”是跨度标签的理想选择，因为它适用于整个跨度，而不适用于特定的时间。 我们可以这样记录：

```
** Go code snippet here **
```

OpenTracing规范为推荐标签提供了称为语义约定的准则。
**Granularity: Spans vs Logs**
日志类似于常规日志语句，它包含时间戳记和一些数据，但与记录日志的跨度相关联。 日志是应用程序或库开发人员完成的检测的一部分。 例如，如果服务器以重定向URL进行响应，则开发人员应记录该URL，因为存在与此类事件相关的明确时间戳。
再次考虑相同的“ Hello Bryan”程序：我们先格式化hello_str，然后再打印它。 这两个操作都需要一定的时间，因此我们可以记录其完成情况：

```
** Go code snippet here **
```
OpenTracing规范为日志字段提供了称为语义约定的准则。
**What about log levels for spans?**
OpenTracing API未指定跨度的日志级别，因为根据应用程序开发人员的需求，它们将有所不同。 但是，库开发人员可以通过将OpenTracing API封装在其中并根据跟踪需求进行模制来创建自己的API，以添加日志级别。

# Context
此处官网尚未更新内容
# Tracers
Tracer Interface
Go Tracer界面创建Span，并了解如何跨过程边界注入（序列化）和提取（反序列化）其元数据。 它具有以下功能：
* Start a new Span
* Inject a SpanContext into a carrier
* Extract a SpanContext from a carrier

这些将在下面更详细地讨论。
**Setting up a Tracer**
**Starting a new Trace**
**Accessing the Active Span**
**Propagating a Trace with Inject/Extract**

为了跨分布式系统中的进程边界进行跟踪，服务需要能够继续由发送每个请求的客户端注入的跟踪。 OpenTracing通过提供注入和提取方法来实现此目标，该方法将跨度的上下文编码为载体。 注入方法允许将SpanContext传递给载体。 提取方法完全相反。 它从载体中提取SpanContext。

**Goroutine-safety**
整个公共API都是goroutine安全的，不需要外部同步。

**适用于实现跟踪系统的API指针**
跟踪系统实现者可以重用或复制粘贴-修改basictracer程序包（在此处找到）。 特别是，请参见basictracer.New（...）。

**API compatibility**
目前，可以在不更改主版本号的情况下进行“轻微”向后不兼容的更改。 随着OpenTracing和opentracing的成熟，向后兼容将成为当务之急。
**Tracer test suite**
线束包中提供了一个测试套件，可以帮助Tracer实现者断言其Tracer工作正常。