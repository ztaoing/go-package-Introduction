# zipkin-go-opentracing 
[https://github.com/openzipkin-contrib/zipkin-go-opentracing](https://github.com/openzipkin-contrib/zipkin-go-opentracing)

用于本机Zipkin跟踪实现Zipkin Go的OpenTracing桥梁。
## Notes
该软件包是一个简单的桥梁，允许OpenTracing API使用者使用Zipkin作为其跟踪后端。
 有关如何使用span和轨迹的详细信息，建议您查看OpenTracing API的文档和自述文件。
对于有兴趣将Zipkin跟踪添加到其Go服务中的开发人员，我们建议您使用Go kit，它是一款出色的工具包，
可用于通过Zipkin来监控您的分布式系统，以及用于将传输，在中间件/工具和业务逻辑等领域进行清晰分离的工具。
## Examples
请检查zipkin-go软件包，以获取有关如何设置Zipkin Go本机跟踪程序的信息。 设置完成后，您可以简单地调用Wrap函数来创建与OpenTracing兼容的网桥。

```
import (
	"github.com/opentracing/opentracing-go"
	"github.com/openzipkin/zipkin-go"
	zipkinhttp "github.com/openzipkin/zipkin-go/reporter/http"
	zipkinot "github.com/openzipkin-contrib/zipkin-go-opentracing"
)

func main() {
	// bootstrap your app...
  
	// zipkin / opentracing specific stuff
	{
		// 建立一个span的reporter
		reporter := zipkinhttp.NewReporter("http://zipkinhost:9411/api/v2/spans")
		defer reporter.Close()
  
		// 创建本地服务的endpoint
		endpoint, err := zipkin.NewEndpoint("myService", "myservice.mydomain.com:80")
		if err != nil {
			log.Fatalf("unable to create local endpoint: %+v\n", err)
		}

		// 初始化链路追踪器
		nativeTracer, err := zipkin.NewTracer(reporter, zipkin.WithLocalEndpoint(endpoint))
		if err != nil {
			log.Fatalf("unable to create tracer: %+v\n", err)
		}

		// 使用 zipkin-go-opentracing 包装 tracer
		tracer := zipkinot.Wrap(nativeTracer)
  
		// 可是选择设置为全局的追踪实例
		opentracing.SetGlobalTracer(tracer)
	}
  
	// do other bootstrapping stuff...
}
```
有关zipkin-go-opentracing的更多信息，请参阅go doc上的文档。

## godoc
#### Variables

```
var Delegator delegatorType
```
Delegator是DelegatingCarrier使用的格式。
#### func Wrap 

```
func Wrap(tr *zipkin.Tracer, opts ...TracerOption) opentracing.Tracer
```
Wrap使用一个zipkin跟踪器并返回一个opentrace的跟踪器
#### type B3InjectOption

```
type B3InjectOption int
```
使用本机OpenTracing HTTPHeadersCarrier时，B3InjectOption类型保存有关B3注入样式的信息。

```
const (
    B3InjectStandard B3InjectOption = iota
    B3InjectSingle
    B3InjectBoth
)
```
可选的B3InjectOption值
#### type DelegatingCarrier

```
type DelegatingCarrier interface {
    State() (model.SpanContext, error)
    SetState(model.SpanContext) error
}
```
DelegatingCarrier是一个灵活的载体接口，可以通过具有存储trace元数据并且已经知道如何进行序列化
#### type FinisherWithDuration

```
type FinisherWithDuration interface {
    FinishedWithDuration(d time.Duration)
}
```
FinisherWithDuration允许以给定的持续时间完成span
#### type SpanContext 

```
type SpanContext model.SpanContext
```
SpanContext保存基本的Span元数据。
#### func (SpanContext) ForeachBaggageItem

```
func (c SpanContext) ForeachBaggageItem(handler func(k, v string) bool)

```
ForeachBaggageItem属于opentracing.SpanContext接口
#### type TracerOption

```
键入TracerOption func（opts * TracerOptions）
```
TracerOption允许使用的功能选项。 参见：http://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis
#### func WithB3InjectOption 

```
func WithB3InjectOption(b3InjectOption B3InjectOption) TracerOption
```
如果使用本机OpenTracing HTTPHeadersCarrier，则WithB3InjectOption设置B3注入样式
#### func WithObserver 

```
func WithObserver(observer otobserver.Observer) TracerOption
```
WithObserver将初始化的观察者分配给opts.observer
#### type TracerOptions

```
type TracerOptions struct {
    // contains filtered or unexported fields
}
```
TracerOptions允许创建自定义的Tracer。