# endpoint
endpoint是许多Go kit组件的基本构建块。 端点由服务器实现，并由客户端调用。

-------

***func Nop***

```
func Nop(context.Context, interface{}) (interface{}, error)
```
Nop不执行任何操作,并返回nil错误。 对测试有用。

-------

***type Endpoint***

```
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```
Endpoint是服务器和客户端的基本构建块。 它代表一个单独的RPC方法。

-------

***type Failer***

```
type Failer interface {
    Failed() error
}
```

Failer可以被包含业务逻辑错误的Go kit响应类型实现。 如果“失败”返回非nil错误，则Go kit传输层可能会将其解释为业务逻辑错误，并且可以对其进行不同于常规的成功响应的方式进行编码。


通常的响应类型不必实现Failer，但对于更复杂的用例可能会有所帮助。 addsvc示例显示了完整的应用程序应如何使用Failer。

-------

***type Middleware***

```
type Middleware func(Endpoint) Endpoint
```
Middleware is a chainable behavior modifier for endpoints.
中间件是一个对endpoints的可连接行为的装饰（装饰者模式）。

-------

***func Chain***

```
func Chain(outer Middleware, others ...Middleware) Middleware
```

Chain是一个能够组织中间件的辅助方法。 请求将按照声明的顺序遍历它们。 也就是说，第一个中间件被视为最外面的中间件（洋葱模式，逐层向中心发展）。


-------

```
package endpoint

import (
	"context"
)

type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)


func Nop(context.Context, interface{}) (interface{}, error) { return struct{}{}, nil }


type Middleware func(Endpoint) Endpoint

func Chain(outer Middleware, others ...Middleware) Middleware {
	return func(next Endpoint) Endpoint {
		for i := len(others) - 1; i >= 0; i-- { // reverse
			next = others[i](next)
		}
		return outer(next)
	}
}


type Failer interface {
	Failed() error
}
```