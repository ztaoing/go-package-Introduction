
*此文使用google翻译完成，并对语义进行了调整*

# Zipkin Library for Go [https://github.com/openzipkin/zipkin-go](https://github.com/openzipkin/zipkin-go)

Zipkin Go是由OpenZipkin社区支持的Zipkin官方的Go语言链路追踪的实现
## packages
zipkin-go的构建考虑到了OpenZipkin社区甚至第3方之间的互操作性，该库由多个软件包组成。
可以在此存储库的根文件夹中找到主要的tracer实现。 Reusable parts not considered core implementation or deemed beneficiary for usage by others are placed in their own packages within this repository.

### model包
该库实现了模型包中提供的Zipkin V2 Span模型。 它包含一个go的数据模型并且与Zipkin V2 API兼容，而且可以从获得到的json自动清理、解析和(反)序列化,他被用作Zipkin官方 V2版本的收集器。

### propagation包
propagation和B3子包，拥有用于链路追踪的，在服务之间传播SpanContext （span标识符和采样标志）的逻辑。 目前，Zipkin B3的传播支持HTTP和GRPC。

### middleware包
中间件子包中包含官方支持的中间件处理程序和跟踪包装器。

*  http
提供了一种易于使用的跟踪服务器端请求的http.Handler中间件。 这使人们可以在使用标准库服务器以及大多数可用的更高级别框架的应用程序中使用此中间件。 一些框架将拥有自己的工具和中间件，以更好地映射其生态系统。
对于HTTP客户端操作，NewTransport可以返回http.RoundTripper，该实现可以包装标准http.Client的Transport或提供的自定义提供的内容，并增加每一个请求的路径追踪。 由于HTTP请求可以具有一个或多个重定向，因此建议始终在*http.Client调用级别或父函数级别周围用一个Span包围HTTP客户端来调用。
为方便起见，提供了NewClient，它返回一个HTTP客户端，该客户端嵌入*http.Client并在调用DoWithAppSpan（）方法时提供围绕HTTP调用的应用程序范围。

* grpc
使用grpc.StatsHandler中间件来跟踪gRPC服务器和客户端请求是很方便的。
对于一个服务器，请在调用NewServer时传递NewServerHandler，例如

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
reporter包中包含各种用Reporter实现的的接口。 它被导出到其自己的程序包中，因为第三方可以在自己的库中使用这些Reporter程序包以构建一个Zipkin生态系统。 zipkin-go跟踪器还使用该接口来应用第三方Reporter的实现。
1. HTTP Reporter
Zipkin用户传输spans到Zipkin服务器时使用的最常见的Reporter类型就是JSON。 Reporter程序拥有一个缓冲区，并异步向后端报告。
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
	// 设置一个 span reporter
	reporter := logreporter.NewReporter(log.New(os.Stderr, "", log.LstdFlags))
	defer reporter.Close()

	// 创建本地服务的 endpoint
	endpoint, err := zipkin.NewEndpoint("myService", "localhost:0")
	if err != nil {
		log.Fatalf("unable to create local endpoint: %+v\n", err)
	}

	// 初始化追踪器
	tracer, err := zipkin.NewTracer(reporter, zipkin.WithLocalEndpoint(endpoint))
	if err != nil {
		log.Fatalf("unable to create tracer: %+v\n", err)
	}

	// 创建一个全局的zipkin http 服务器中间件
	serverMiddleware := zipkinhttp.NewServerMiddleware(
		tracer, zipkinhttp.TagResponseSize(true),
	)

	// 创建一个全局的zipkin http 客户端
	client, err := zipkinhttp.NewClient(tracer, zipkinhttp.ClientTrace(true))
	if err != nil {
		log.Fatalf("unable to create client: %+v\n", err)
	}

	// 初始化路由
	router := mux.NewRouter()

	// 使用zipkin http服务器中间件开启web服务
	ts := httptest.NewServer(serverMiddleware(router))
	defer ts.Close()

	// 建立 handlers
	router.Methods("GET").Path("/some_function").HandlerFunc(someFunc(client, ts.URL))
	router.Methods("POST").Path("/other_function").HandlerFunc(otherFunc(client))

	// 发起调用一个具体的方法
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

		//从上下中找回被服务器中间件创建的span 
		span := zipkin.SpanFromContext(r.Context())
		span.Tag("custom_key", "some value")

		// 做一些耗时的操作
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

