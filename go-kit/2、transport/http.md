# http


-------
client.go
```golang
package http

import (
	"bytes"
	"context"
	"encoding/json"
	"encoding/xml"
	"io"
	"io/ioutil"
	"net/http"
	"net/url"

	"github.com/go-kit/kit/endpoint"
)

// HTTPClient是模拟* http.Client的接口。
type HTTPClient interface {
	Do(req *http.Request) (*http.Response, error)
}

// Client包装了一个URL，并提供一种实现endpoint.Endpoint的方法。
type Client struct {
	client         HTTPClient
	req            CreateRequestFunc
	dec            DecodeResponseFunc
	before         []RequestFunc
	after          []ClientResponseFunc
	finalizer      []ClientFinalizerFunc
	bufferedStream bool
}

// NewClient为单个远程方法构造一个可用的客户端。
func NewClient(method string, tgt *url.URL, enc EncodeRequestFunc, dec DecodeResponseFunc, options ...ClientOption) *Client {
	return NewExplicitClient(makeCreateRequestFunc(method, tgt, enc), dec, options...)
}

//NewExplicitClient与NewClient类似，但使用CreateRequestFunc代替方法，目标URL和EncodeRequestFunc，从而可以更好地控制发出的HTTP请求。
func NewExplicitClient(req CreateRequestFunc, dec DecodeResponseFunc, options ...ClientOption) *Client {
	c := &Client{
		client: http.DefaultClient,
		req:    req,
		dec:    dec,
	}
	for _, option := range options {
		option(c)
	}
	return c
}

// ClientOption为客户端设置一个可选参数。
type ClientOption func(*Client)

// SetClient设置用于请求的基础HTTP客户端。
// 默认情况下使用 http.DefaultClient
func SetClient(client HTTPClient) ClientOption {
	return func(c *Client) { c.client = client }
}

// 在发送HTTP请求之前调用ClientBefore执行一个或多个RequestFuncs。
func ClientBefore(before ...RequestFunc) ClientOption {
	return func(c *Client) { c.before = append(c.before, before...) }
}

// ClientAfter添加一个或多个ClientResponseFuncs，它们在解码之前将应用于传入的HTTP响应。 这对于从响应中获取任何内容并将其添加到解码之前的上下文中，是很有用处的。
func ClientAfter(after ...ClientResponseFunc) ClientOption {
	return func(c *Client) { c.after = append(c.after, after...) }
}

// ClientFinalizer添加一个或多个ClientFinalizerFunc，以在每个HTTP请求的末尾执行。 Finalizers按其添加的顺序执行。 默认情况下，不会注册finalizer。
func ClientFinalizer(f ...ClientFinalizerFunc) ClientOption {
	return func(s *Client) { s.finalizer = append(s.finalizer, f...) }
}

BufferedStream设置HTTP响应主体是否保持打开状态，以便以后可以读取。 对于将文件作为缓冲流传输很有用。 该主体必须用完并关闭才能正确结束请求。
func BufferedStream(buffered bool) ClientOption {
	return func(c *Client) { c.bufferedStream = buffered }
}

//Endpoint返回一个可用的Go kit Endpoint，该Endpoint调用远程HTTP的Endpoint。 
func (c Client) Endpoint() endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		ctx, cancel := context.WithCancel(ctx)

		var (
			resp *http.Response
			err  error
		)
		if c.finalizer != nil {
			defer func() {
				if resp != nil {
					ctx = context.WithValue(ctx, ContextKeyResponseHeaders, resp.Header)
					ctx = context.WithValue(ctx, ContextKeyResponseSize, resp.ContentLength)
				}
				for _, f := range c.finalizer {
					f(ctx, err)
				}
			}()
		}

		req, err := c.req(ctx, request)
		if err != nil {
			cancel()
			return nil, err
		}

		for _, f := range c.before {
			ctx = f(ctx, req)
		}

		resp, err = c.client.Do(req.WithContext(ctx))
		if err != nil {
			cancel()
			return nil, err
		}

		//如果调用方请求一个缓冲流，则在endpoint返回时我们不会取消上下文。 相反，我们应该在关闭response的body的时候调用cancel函数。
	if c.bufferedStream {
			resp.Body = bodyWithCancel{ReadCloser: resp.Body, cancel: cancel}
		} else {
			defer resp.Body.Close()
			defer cancel()
		}

		for _, f := range c.after {
			ctx = f(ctx, resp)
		}

		response, err := c.dec(ctx, resp)
		if err != nil {
			return nil, err
		}

		return response, nil
	}
}

// bodyWithCancel是io.ReadCloser的一个包装，当关闭的时候cancel函数会被调用
type bodyWithCancel struct {
	io.ReadCloser

	cancel context.CancelFunc
}

func (bwc bodyWithCancel) Close() error {
	bwc.ReadCloser.Close()
	bwc.cancel()
	return nil
}

//在返回响应后，可以使用ClientFinalizerFunc在客户端HTTP请求的末尾执行工作。 主要用途是用于错误记录。
// 上下文中在带有ContextKeyResponse前缀的键下提供了其他响应参数。 
//注意：err可能为nil。 根据何时发生错误，有可能也没有其他响应参数。
type ClientFinalizerFunc func(ctx context.Context, err error)


//EncodeJSONRequest是一个EncodeRequestFunc，它将请求序列化为JSON对象，并存储到Request的body中。 
//许多基于HTTP的JSON服务都可以将其用作明智的默认设置。 如果请求实现了Headerer，则将提供的headers应用到请求中。
func EncodeJSONRequest(c context.Context, r *http.Request, request interface{}) error {
	r.Header.Set("Content-Type", "application/json; charset=utf-8")
	if headerer, ok := request.(Headerer); ok {
		for k := range headerer.Headers() {
			r.Header.Set(k, headerer.Headers().Get(k))
		}
	}
	var b bytes.Buffer
	r.Body = ioutil.NopCloser(&b)
	return json.NewEncoder(&b).Encode(request)
}


//EncodeXMLRequest是一个EncodeRequestFunc，它将请求序列化为XML对象保存到Request的body中。
//如果请求实现了Headerer，则将提供的headers应用到请求中。
func EncodeXMLRequest(c context.Context, r *http.Request, request interface{}) error {
	r.Header.Set("Content-Type", "text/xml; charset=utf-8")
	if headerer, ok := request.(Headerer); ok {
		for k := range headerer.Headers() {
			r.Header.Set(k, headerer.Headers().Get(k))
		}
	}
	var b bytes.Buffer
	r.Body = ioutil.NopCloser(&b)
	return xml.NewEncoder(&b).Encode(request)
}

//
//
//

func makeCreateRequestFunc(method string, target *url.URL, enc EncodeRequestFunc) CreateRequestFunc {
	return func(ctx context.Context, request interface{}) (*http.Request, error) {
		req, err := http.NewRequest(method, target.String(), nil)
		if err != nil {
			return nil, err
		}

		if err = enc(ctx, req, request); err != nil {
			return nil, err
		}

		return req, nil
	}
}
```

-------
encode_decode.go

```golang
package http

import (
	"context"
	"net/http"
)


//DecodeRequestFunc从HTTP请求对象中提取用户域请求对象。 它被设计成用在HTTP服务器中的服务器端的endpoints。 
//一个简单的DecodeRequestFunc可以是JSON从请求的body中解码为具体的请求类型。
type DecodeRequestFunc func(context.Context, *http.Request) (request interface{}, err error)


//EncodeRequestFunc将传递的请求对象编码为HTTP请求对象。 它被设计用在HTTP客户端，为了客户端的endpoints。 一个简单的EncodeRequestFunc可以将JSON对象直接编码到请求的body中。
type EncodeRequestFunc func(context.Context, *http.Request, interface{}) error


//CreateRequestFunc基于传递的请求对象创建一个发送的HTTP请求。 它被设计用在HTTP客户端，为了客户端的endpoints。 
//它是EncodeRequestFunc的更强大的版本，如果需要对HTTP请求进行更细粒度的控制，则可以使用它。
type CreateRequestFunc func(context.Context, interface{}) (*http.Request, error)


//EncodeResponseFunc将响应对象编码为HTTPresponse writer。 它被设计用在HTTP服务器中的服务器的endpoints。
// 一个简单的EncodeResponseFunc可以将JSON对象直接编码到响应的body中。
type EncodeResponseFunc func(context.Context, http.ResponseWriter, interface{}) error

// DecodeResponseFunc extracts a user-domain response object from an HTTP
// response object. It's designed to be used in HTTP clients, for client-side
// endpoints. One straightforward DecodeResponseFunc could be something that
// JSON decodes from the response body to the concrete response type.

//DecodeResponseFunc从HTTP响应对象中提取用户域响应对象。 它被设计用在HTTP客户端，为了客户端的endpoints。
// 一个简单的DecodeResponseFunc可以将JSON对象从响应主体解码为具体响应类型。
type DecodeResponseFunc func(context.Context, *http.Response) (response interface{}, err error)
```
request_response_funcs.go 

```golang
package http

import (
	"context"
	"net/http"
)



//RequestFunc可以从HTTP请求中获取信息，并将其放入请求上下文中。
//在服务器中，在调用endpoint之前执行RequestFuncs。 
//在客户端中，在创建请求之后但在调用HTTP客户端之前，将执行RequestFuncs。
type RequestFunc func(context.Context, *http.Request) context.Context



//ServerResponseFunc可以从请求上下文中获取信息，并使用它来操作ResponseWriter。
//ServerResponseFuncs仅在服务器中执行，并且要在在调用endpoint之后，而且在写入响应之前执行。
type ServerResponseFunc func(context.Context, http.ResponseWriter) context.Context

// ClientResponseFunc may take information from an HTTP request and make the
// response available for consumption. ClientResponseFuncs are only executed in
// clients, after a request has been made, but prior to it being decoded.

//ClientResponseFunc可以从HTTP请求中获取信息，并使响应可用来使用。 
//ClientResponseFuncs仅在在客户端中执行，但是要在发出请求之后，而且在解码之前执行。
type ClientResponseFunc func(context.Context, *http.Response) context.Context


//SetContentType返回一个ServerResponseFunc，它将Content-Type标头设置为传入的值。
func SetContentType(contentType string) ServerResponseFunc {
	return SetResponseHeader("Content-Type", contentType)
}


//Set Response Header返回设置为给定header的ServerResponseFunc。
func SetResponseHeader(key, val string) ServerResponseFunc {
	return func(ctx context.Context, w http.ResponseWriter) context.Context {
		w.Header().Set(key, val)
		return ctx
	}
}

// Set Request Header returns a RequestFunc that sets the given header.
func SetRequestHeader(key, val string) RequestFunc {
	return func(ctx context.Context, r *http.Request) context.Context {
		r.Header.Set(key, val)
		return ctx
	}
}

//PopulateRequestContext是一个RequestFunc，它将几个值从HTTP请求填充到上下文中。 这些值可以使用此包中的相应的ContextKey类型提取出来。
func PopulateRequestContext(ctx context.Context, r *http.Request) context.Context {
	for k, v := range map[contextKey]string{
	   //Method
		ContextKeyRequestMethod:          r.Method,
		//URI
		ContextKeyRequestURI:             r.RequestURI,
		//Path
		ContextKeyRequestPath:            r.URL.Path,
		//Proto
		ContextKeyRequestProto:           r.Proto,
		//Host
		ContextKeyRequestHost:            r.Host,
		//RemoteAddr
		ContextKeyRequestRemoteAddr:      r.RemoteAddr,
		//XForwardedFor
		ContextKeyRequestXForwardedFor:   r.Header.Get("X-Forwarded-For"),
		//XForwardedProto
		ContextKeyRequestXForwardedProto: r.Header.Get("X-Forwarded-Proto"),
		//Authorization
		ContextKeyRequestAuthorization:   r.Header.Get("Authorization"),
		//Referer
		ContextKeyRequestReferer:         r.Header.Get("Referer"),
		//UserAgent
		ContextKeyRequestUserAgent:       r.Header.Get("User-Agent"),
		//XRequestID
		ContextKeyRequestXRequestID:      r.Header.Get("X-Request-Id"),
		//Accept
		ContextKeyRequestAccept:          r.Header.Get("Accept"),
	} {
		ctx = context.WithValue(ctx, k, v)
	}
	return ctx
}

type contextKey int

const (
	// ContextKeyRequestMethod 通过PopulateRequestContext填充在上下文中	// 他的值是 r.Method.
	ContextKeyRequestMethod contextKey = iota

	// ContextKeyRequestURI is populated in the context by
	// PopulateRequestContext. 
	// 他的值是 r.RequestURI.
	ContextKeyRequestURI

	// ContextKeyRequestPath is populated in the context by
	// PopulateRequestContext. 他的值是 r.URL.Path.
	ContextKeyRequestPath

	// ContextKeyRequestProto is populated in the context by
	// PopulateRequestContext. 他的值是 r.Proto.
	ContextKeyRequestProto

	// ContextKeyRequestHost is populated in the context by
	// PopulateRequestContext. 他的值是 r.Host.
	ContextKeyRequestHost

	// ContextKeyRequestRemoteAddr is populated in the context by
	// PopulateRequestContext. 他的值是 r.RemoteAddr.
	ContextKeyRequestRemoteAddr

	// ContextKeyRequestXForwardedFor is populated in the context by
	// PopulateRequestContext. 他的值是 r.Header.Get("X-Forwarded-For").
	ContextKeyRequestXForwardedFor

	// ContextKeyRequestXForwardedProto is populated in the context by
	// PopulateRequestContext. 他的值是 r.Header.Get("X-Forwarded-Proto").
	ContextKeyRequestXForwardedProto

	// ContextKeyRequestAuthorization is populated in the context by
	// PopulateRequestContext. 他的值是 r.Header.Get("Authorization").
	ContextKeyRequestAuthorization

	// ContextKeyRequestReferer is populated in the context by
	// PopulateRequestContext. 他的值是 r.Header.Get("Referer").
	ContextKeyRequestReferer

	// ContextKeyRequestUserAgent is populated in the context by
	// PopulateRequestContext. 他的值是 r.Header.Get("User-Agent").
	ContextKeyRequestUserAgent

	// ContextKeyRequestXRequestID is populated in the context by
	// PopulateRequestContext. 他的值是 r.Header.Get("X-Request-Id").
	ContextKeyRequestXRequestID

	// ContextKeyRequestAccept is populated in the context by
	// PopulateRequestContext. 他的值是 r.Header.Get("Accept").
	ContextKeyRequestAccept

	// ContextKeyResponseHeaders is populated in the context whenever a
	// ServerFinalizerFunc is specified. 他的值是 of type http.Header, and
	// is captured only once the entire response has been written.
	ContextKeyResponseHeaders

	// ContextKeyResponseSize is populated in the context whenever a
	// ServerFinalizerFunc is specified. 他的值是  int64 类型.
	ContextKeyResponseSize
)
```
server.go

```golang
package http

import (
	"context"
	"encoding/json"
	"net/http"

	"github.com/go-kit/kit/endpoint"
	"github.com/go-kit/kit/log"
	"github.com/go-kit/kit/transport"
)

// Server 包装了一个endpoint同时实现了 http.Handler.
type Server struct {
	e            endpoint.Endpoint
	dec          DecodeRequestFunc
	enc          EncodeResponseFunc
	before       []RequestFunc
	after        []ServerResponseFunc
	errorEncoder ErrorEncoder
	finalizer    []ServerFinalizerFunc
	errorHandler transport.ErrorHandler
}

// NewServer 构造了一个 server,它实现了http.Handler ，同时包装了被提供endpoint.
func NewServer(
	e endpoint.Endpoint,
	dec DecodeRequestFunc,
	enc EncodeResponseFunc,
	options ...ServerOption,
) *Server {
	s := &Server{
		e:            e,
		dec:          dec,
		enc:          enc,
		errorEncoder: DefaultErrorEncoder,
		errorHandler: transport.NewLogErrorHandler(log.NewNopLogger()),
	}
	for _, option := range options {
		option(s)
	}
	return s
}

//ServerOption为Server设置一个可选参数。
type ServerOption func(*Server)

// ServerBefore functions are executed on the HTTP request object before the request is decoded.
func ServerBefore(before ...RequestFunc) ServerOption {
	return func(s *Server) { s.before = append(s.before, before...) }
}

// ServerAfter functions are executed on the HTTP response writer after the endpoint is invoked, but before anything is written to the client.
func ServerAfter(after ...ServerResponseFunc) ServerOption {
	return func(s *Server) { s.after = append(s.after, after...) }
}


//ServerErrorEncoder用于在处理请求时遇到错误时对http.ResponseWriter进行编码。 客户可以使用它来提供自定义错误格式和响应代码。 
//默认情况下,将使用DefaultErrorEncoder。
func ServerErrorEncoder(ee ErrorEncoder) ServerOption {
	return func(s *Server) { s.errorEncoder = ee }
}



//ServerErrorLogger用于记录非终端错误。 默认情况下，不记录任何错误。 这旨在作为一种诊断措施。 
//应该在自定义的ServerErrorEncoder或ServerFinalizer中执行对错误处理的更细粒度的控制，包括更详细的日志记录，这两者都可以访问上下文。
// 已弃用：改用ServerErrorHandler。
func ServerErrorLogger(logger log.Logger) ServerOption {
	return func(s *Server) { s.errorHandler = transport.NewLogErrorHandler(logger) }
}


//ServerErrorHandler用于处理非终端错误。 默认情况下，将忽略非终端错误。 这旨在作为一种诊断措施。 
//应该在自定义的ServerErrorEncoder或ServerFinalizer中执行对错误处理的更细粒度的控制，包括更详细的日志记录，这两者都可以访问上下文。
func ServerErrorHandler(errorHandler transport.ErrorHandler) ServerOption {
	return func(s *Server) { s.errorHandler = errorHandler }
}


//ServerFinalizer在每个HTTP请求的末尾执行。
//  默认情况下，没有注册finalizer。
func ServerFinalizer(f ...ServerFinalizerFunc) ServerOption {
	return func(s *Server) { s.finalizer = append(s.finalizer, f...) }
}

// **ServeHTTP 实现了 http.Handler.**
func (s Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	if len(s.finalizer) > 0 {
		iw := &interceptingWriter{w, http.StatusOK, 0}
		defer func() {
			ctx = context.WithValue(ctx, ContextKeyResponseHeaders, iw.Header())
			ctx = context.WithValue(ctx, ContextKeyResponseSize, iw.written)
			//逐个执行finalizer
			for _, f := range s.finalizer {
				f(ctx, iw.code, r)
			}
		}()
		w = iw
	}
//在请求之前
	for _, f := range s.before {
		ctx = f(ctx, r)
	}
//解码
	request, err := s.dec(ctx, r)
	if err != nil {
		s.errorHandler.Handle(ctx, err)
		s.errorEncoder(ctx, err, w)
		return
	}
//endpoint
	response, err := s.e(ctx, request)
	if err != nil {
		s.errorHandler.Handle(ctx, err)
		s.errorEncoder(ctx, err, w)
		return
	}
//在之后
	for _, f := range s.after {
		ctx = f(ctx, w)
	}
//编码
	if err := s.enc(ctx, w, response); err != nil {
		s.errorHandler.Handle(ctx, err)
		s.errorEncoder(ctx, err, w)
		return
	}
}

// 鼓励用户使用自定义ErrorEncoders将HTTP错误编码到其客户端，并且可能希望传递并检查自己的错误类型。 请参阅示例shipping/handling服务。
type ErrorEncoder func(ctx context.Context, err error, w http.ResponseWriter)


//将响应写入客户端后，可以使用ServerFinalizerFunc在HTTP请求结束时执行工作。 主要用途是用于请求日志记录。 
//除了功能签名中提供的响应代码外，上下文中还提供了其他响应参数，这些参数带有ContextKeyResponse前缀。
type ServerFinalizerFunc func(ctx context.Context, code int, r *http.Request)


//NopRequestDecoder是一个DecodeRequestFunc，可用于不需要解码的请求，仅返回nil，nil就可以。
func NopRequestDecoder(ctx context.Context, r *http.Request) (interface{}, error) {
	return nil, nil
}

// EncodeJSONResponse is a EncodeResponseFunc that serializes the response as a
// JSON object to the ResponseWriter. Many JSON-over-HTTP services can use it as
// a sensible default. If the response implements Headerer, the provided headers
// will be applied to the response. If the response implements StatusCoder, the
// provided StatusCode will be used instead of 200.

//EncodeJSONResponse是一个EncodeResponseFunc，它将响应序列作为JSON对象并序列化到ResponseWriter。 许多基于HTTP的JSON服务都可以将其用作推荐的默认设置。
// 如果响应实现了Headerer，则提供的headers将应用在响应中。 如果响应实现了StatusCoder，则将使用提供的StatusCode代替200。
func EncodeJSONResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	if headerer, ok := response.(Headerer); ok {
		for k, values := range headerer.Headers() {
			for _, v := range values {
				w.Header().Add(k, v)
			}
		}
	}
	code := http.StatusOK
	if sc, ok := response.(StatusCoder); ok {
		code = sc.StatusCode()
	}
	w.WriteHeader(code)
	if code == http.StatusNoContent {
		return nil
	}
	return json.NewEncoder(w).Encode(response)
}

// DefaultErrorEncoder writes the error to the ResponseWriter, by default a
// content type of text/plain, a body of the plain text of the error, and a
// status code of 500. If the error implements Headerer, the provided headers
// will be applied to the response. If the error implements json.Marshaler, and
// the marshaling succeeds, a content type of application/json and the JSON
// encoded form of the error will be used. If the error implements StatusCoder,
// the provided StatusCode will be used instead of 500.

//DefaultErrorEncoder将错误写入ResponseWriter，默认情况下为text/plain的内容类型，错误的纯文本body以及状态码500。
//如果error实现Headerer，则提供的headers将应用在响应 。
//如果error实现json.Marshaler，并且marshal成功，则将使用application/json的内容类型和错误的JSON编码形式。
// 如果error实现了StatusCoder，则将使用提供的StatusCode代替500。
func DefaultErrorEncoder(_ context.Context, err error, w http.ResponseWriter) {
	contentType, body := "text/plain; charset=utf-8", []byte(err.Error())
	if marshaler, ok := err.(json.Marshaler); ok {
		if jsonBody, marshalErr := marshaler.MarshalJSON(); marshalErr == nil {
			contentType, body = "application/json; charset=utf-8", jsonBody
		}
	}
	w.Header().Set("Content-Type", contentType)
	if headerer, ok := err.(Headerer); ok {
		for k, values := range headerer.Headers() {
			for _, v := range values {
				w.Header().Add(k, v)
			}
		}
	}
	code := http.StatusInternalServerError
	if sc, ok := err.(StatusCoder); ok {
		code = sc.StatusCode()
	}
	w.WriteHeader(code)
	w.Write(body)
}



//StatusCoder由DefaultErrorEncoder检查。 如果error实现了StatusCoder，则在编码错误时将使用StatusCode。
// 默认情况下，使用StatusInternalServerError（500）。
type StatusCoder interface {
	StatusCode() int
}

// Headerer is checked by DefaultErrorEncoder. If an error value implements
// Headerer, the provided headers will be applied to the response writer, after
// the Content-Type is set.

//Headerer由DefaultErrorEncoder来检查。 
//如果error value 实现了Headerer，则在设置Content-Type之后，提供的headers将应用在response writer中。
type Headerer interface {
	Headers() http.Header
}

type interceptingWriter struct {
	http.ResponseWriter
	code    int
	written int64
}

//无法显式调用WriteHeader，因此必须注意将w.code初始化为其默认值http.StatusOK。
func (w *interceptingWriter) WriteHeader(code int) {
	w.code = code
	w.ResponseWriter.WriteHeader(code)
}

func (w *interceptingWriter) Write(p []byte) (int, error) {
	n, err := w.ResponseWriter.Write(p)
	w.written += int64(n)
	return n, err
}
```