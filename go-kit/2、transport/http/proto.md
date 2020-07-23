# http/proto

import "github.com/go-kit/kit/transport/http/proto"

client.go
```
package proto

import (
	"bytes"
	"context"
	"errors"
	"io/ioutil"
	"net/http"

	httptransport "github.com/go-kit/kit/transport/http"
	"github.com/golang/protobuf/proto"
)

// EncodeProtoRequest 把请求序列化为Protobuf格式
// 如果情趣实现了Headerer标头，那么将会应用提供的header到请求中。
//如果给出的请求没有实现proto.Message，那么会返回一个错误

func EncodeProtoRequest(_ context.Context, r *http.Request, preq interface{}) error {
	r.Header.Set("Content-Type", "application/x-protobuf")
	if headerer, ok := preq.(httptransport.Headerer); ok {
		for k := range headerer.Headers() {
			r.Header.Set(k, headerer.Headers().Get(k))
		}
	}
	req, ok := preq.(proto.Message)
	if !ok {
		return errors.New("response does not implement proto.Message")
	}

	b, err := proto.Marshal(req)
	if err != nil {
		return err
	}
	r.ContentLength = int64(len(b))
	r.Body = ioutil.NopCloser(bytes.NewReader(b))
	return nil
}
```

server.go

```
package proto

import (
	"context"
	"errors"
	"net/http"

	httptransport "github.com/go-kit/kit/transport/http"
	"github.com/golang/protobuf/proto"
)

// EncodeProtoResponse 把应答序列化为Protobuf格式
许多以HTTP为基础的Proto服务，可以使用EncodeProtoResponse作为一个推荐的默认配置。 如果响应实现Headerer，则提供的标头将应用于响应。 如果响应实现了StatusCoder，则将使用提供的StatusCode代替200。
func EncodeProtoResponse(ctx context.Context, w http.ResponseWriter, pres interface{}) error {
	res, ok := pres.(proto.Message)
	if !ok {
		return errors.New("response does not implement proto.Message")
	}
	w.Header().Set("Content-Type", "application/x-protobuf")
	if headerer, ok := w.(httptransport.Headerer); ok {
		for k := range headerer.Headers() {
			w.Header().Set(k, headerer.Headers().Get(k))
		}
	}
	code := http.StatusOK
	if sc, ok := pres.(httptransport.StatusCoder); ok {
		code = sc.StatusCode()
	}
	w.WriteHeader(code)
	if code == http.StatusNoContent {
		return nil
	}
	if res == nil {
		return nil
	}
	b, err := proto.Marshal(res)
	if err != nil {
		return err
	}
	_, err = w.Write(b)
	if err != nil {
		return err
	}
	return nil
}
```