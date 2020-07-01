
*此文使用google翻译完成，并对语义进行了调整*

# Zipkin Library for Go

Zipkin Go是OpenZipkin社区支持的Zipkin的官方Go Tracer的实现
## packages
zipkin-go的构建考虑到了OpenZipkin社区甚至第3方之间的互操作性，该库由多个软件包组成。
可以在此存储库的根文件夹中找到主要的跟踪实现。 Reusable parts not considered core implementation or deemed beneficiary for usage by others are placed in their own packages within this repository.

### model包
该库实现了模型包中提供的Zipkin V2 Span模型。 它包含一个与Zipkin V2 API兼容的Go数据模型，并且可以自动清理，解析和解序列化官方Zipkin V2收集器所使用的所需JSON表示形式。
### propagation包
propagation和B3子包包含用于在参与跟踪的服务之间传播SpanContext（跨度标识符和采样标志）的逻辑。 目前，HTTP和GRPC支持Zipkin B3传播。
### middleware包
中间件子包包含官方支持的中间件处理程序和跟踪包装器。

*  http
提供了一种易于使用的http.Handler中间件来跟踪服务器端请求。 这使人们可以在使用标准库服务器以及大多数可用的更高级别框架的应用程序中使用此中间件。 一些框架将拥有自己的工具和中间件，以更好地映射其生态系统。
对于HTTP客户端操作，NewTransport可以返回http.RoundTripper实现，该实现可以包装标准http.Client的Transport或提供的自定义提供的内容，并按请求跟踪添加。 由于HTTP请求可以具有一个或多个重定向，因此建议始终在* http.Client调用级别或父函数级别周围用一个Span包围HTTP客户端调用。
为方便起见，提供了NewClient，它返回一个HTTP客户端，该客户端嵌入* http.Client并在调用DoWithAppSpan（）方法时提供围绕HTTP调用的应用程序范围。
* grpc
易于使用的grpc.StatsHandler中间件用于跟踪gRPC服务器和客户端请求。
对于服务器，请在调用NewServer时传递NewServerHandler，例如

```
import (
	"google.golang.org/grpc"
	zipkingrpc "github.com/openzipkin/zipkin-go/middleware/grpc"
)

server = grpc.NewServer(grpc.StatsHandler(zipkingrpc.NewServerHandler(tracer)))
```
对于客户端，请在致电Dial时传递NewClientHandler，例如

```
import (
	"google.golang.org/grpc"
	zipkingrpc "github.com/openzipkin/zipkin-go/middleware/grpc"
)

conn, err = grpc.Dial(addr, grpc.WithStatsHandler(zipkingrpc.NewClientHandler(tracer)))
```
* reporter
报告程序包包含各种Reporter实现使用的接口。 它被导出到其自己的程序包中，因为第三方可以使用它们在其自己的库中使用这些Reporter程序包以导出到Zipkin生态系统。 zipkin-go跟踪器还使用该接口来接受第三方Reporter实现。
1. HTTP Reporter
Zipkin用户使用HTTP上的JSON将Spans传输到Zipkin服务器时使用的最常见的Reporter类型。 报告程序拥有一个缓冲区，并异步向后端报告。
2. Kafka Reporter
高性能Reporter使用Kafka Producer消化JSON V2 Spans来将Span传输到Zipkin服务器。 记者在下面使用Sarama异步制作器。

#### usage and examples

```
// Copyright 2019 The OpenZipkin Authors
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

package zipkin_test

import (
	"log"
	"net/http"
	"net/http/httptest"
	"os"
	"time"

	"github.com/gorilla/mux"

	zipkin "github.com/openzipkin/zipkin-go"
	zipkinhttp "github.com/openzipkin/zipkin-go/middleware/http"
	logreporter "github.com/openzipkin/zipkin-go/reporter/log"
)

func Example() {
	// set up a span reporter
	reporter := logreporter.NewReporter(log.New(os.Stderr, "", log.LstdFlags))
	defer reporter.Close()

	// create our local service endpoint
	endpoint, err := zipkin.NewEndpoint("myService", "localhost:0")
	if err != nil {
		log.Fatalf("unable to create local endpoint: %+v\n", err)
	}

	// initialize our tracer
	tracer, err := zipkin.NewTracer(reporter, zipkin.WithLocalEndpoint(endpoint))
	if err != nil {
		log.Fatalf("unable to create tracer: %+v\n", err)
	}

	// create global zipkin http server middleware
	serverMiddleware := zipkinhttp.NewServerMiddleware(
		tracer, zipkinhttp.TagResponseSize(true),
	)

	// create global zipkin traced http client
	client, err := zipkinhttp.NewClient(tracer, zipkinhttp.ClientTrace(true))
	if err != nil {
		log.Fatalf("unable to create client: %+v\n", err)
	}

	// initialize router
	router := mux.NewRouter()

	// start web service with zipkin http server middleware
	ts := httptest.NewServer(serverMiddleware(router))
	defer ts.Close()

	// set-up handlers
	router.Methods("GET").Path("/some_function").HandlerFunc(someFunc(client, ts.URL))
	router.Methods("POST").Path("/other_function").HandlerFunc(otherFunc(client))

	// initiate a call to some_func
	req, err := http.NewRequest("GET", ts.URL+"/some_function", nil)
	if err != nil {
		log.Fatalf("unable to create http request: %+v\n", err)
	}

	res, err := client.DoWithAppSpan(req, "some_function")
	if err != nil {
		log.Fatalf("unable to do http request: %+v\n", err)
	}
	res.Body.Close()

	// Output:
}

func someFunc(client *zipkinhttp.Client, url string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		log.Printf("some_function called with method: %s\n", r.Method)

		// retrieve span from context (created by server middleware)
		span := zipkin.SpanFromContext(r.Context())
		span.Tag("custom_key", "some value")

		// doing some expensive calculations....
		time.Sleep(25 * time.Millisecond)
		span.Annotate(time.Now(), "expensive_calc_done")

		newRequest, err := http.NewRequest("POST", url+"/other_function", nil)
		if err != nil {
			log.Printf("unable to create client: %+v\n", err)
			http.Error(w, err.Error(), 500)
			return
		}

		ctx := zipkin.NewContext(newRequest.Context(), span)

		newRequest = newRequest.WithContext(ctx)

		res, err := client.DoWithAppSpan(newRequest, "other_function")
		if err != nil {
			log.Printf("call to other_function returned error: %+v\n", err)
			http.Error(w, err.Error(), 500)
			return
		}
		res.Body.Close()
	}
}

func otherFunc(client *zipkinhttp.Client) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		log.Printf("other_function called with method: %s\n", r.Method)
		time.Sleep(50 * time.Millisecond)
	}
}

```

