# 序言
在设计一个框架之前，需要明白框架核心为我们解决了什么问题，这样才能想明白需要在框架中实现什么功能。
golang的标准库`net/http`提供了简单的Web功能，即监听端口、映射动态路由、解析HTTP报文。一些web开发中简单的需求并不支持，需要自行实现。
# HTTP基础
## 标准库 net/http 的使用（标准库启动Web服务）
通过例子理解标准库`net/http`的使用
```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", indexHandler)
	http.HandleFunc("/Hello", helloHandler)
	log.Fatal(http.ListenAndServe(":9999", nil))
}

// handler echoes r.URL.Path
func indexHandler(w http.ResponseWriter, req *http.Request) {
}

// handler echoes r.URL.Header
func helloHandler(w http.ResponseWriter, req *http.Request) {
	for k, v := range req.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
}
```
- 代码中设置了两个路由`/`和`/Hello`，分别绑定 *indexHandler* 和 *helloHandler*，根据不同的http请求调用不同的处理函数。
  - 访问`/`，响应是`URL.Path = /`
  - 访问`/Hello`，响应是请求头（Header）中的键值对信息。
- `main`函数的最后一行`log.Fatal(http.ListenAndServe(":9999", nil)`用于启动Web服务。
  - 第一个参数是地址，`:9999`表示在 *9999* 端口监听。
  - 第二个参数代表处理所有的HTTP请求的实例，`nil`代表使用标准库中的实例处理。第二个参数是基于`net/http`标准库实现Web框架的入口。

## 实现http.Handler接口
```go
package http

type Handler interface {
	ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```
第二个参数的类型是什么呢？通过查看`net/http`的源码可以发现，Handler是一个接口，需要实现方法 *ServeHTTP* ，也就是说，只要传入任何实现了 *ServerHTTP* 接口的实例，所有的HTTP请求，就都交给了该实例处理了。
```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

// Engine is the uni handler for all requests
type Engine struct{}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/": // handler echoes r.URL.Path
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	case "/hello": // handler echoes r.URL.Header
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	default:
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}

func main() {
	engine := new(Engine)
	log.Fatal(http.ListenAndServe(":9999", engine))
}
```
- 用`switch...case...`把分开写的两个路由整合到了一起
- 定义一个空结构体`Engine`，实现了方法`ServeHTTP`。
  - 这个方法有2个参数，第一个参数是 *ResponseWriter* ，利用 *ResponseWriter* 可以构造针对该请求的响应；第二个参数是 *Request* ，该对象包含了该HTTP请求的所有的信息，比如请求地址、Header和Body等信息。
- 在 `main` 函数中，我们给 *ListenAndServe* 方法的第二个参数传入了刚才创建的`engine`实例。
  至此，我们走出了实现Web框架的第一步，即，将所有的HTTP请求转向了我们自己的处理逻辑。
  在实现`Engine`之前，我们调用 *http.HandleFunc* 实现了路由和Handler的映射，也就是只能针对具体的路由写处理逻辑。比如`/hello`。但是在实现`Engine`之后，我们拦截了所有的HTTP请求，拥有了统一的控制入口。在这里我们可以自由定义路由映射的规则，也可以统一添加一些处理逻辑，例如日志、异常处理等。
- 代码的运行结果与之前的是一致的。
## Gee框架的雏形
重新组织上面的代码，搭建出整个框架的雏形
最终的代码目录结构：
```txt
gee/
  |--gee.go
  |--go.mod
main.go
go.mod
```
Gee框架的雏形实现了路由映射表，提供了用户注册静态路由的方法，包装了启动服务的函数
### go.mod
创建`go.mod`文件，在`go.mod`中使用`replace`将 gee 指向 `./gee`
```txt
module example

go 1.20.2

require gee v0.0.0

replace gee => ./gee
```
### main.go
打开`main.go`文件，写入以下代码：
使用`New()`创建 gee 的实例，使用 `GET()`方法添加路由，最后使用`Run()`启动Web服务。这里的路由，只是静态路由，不支持`/hello/:name`这样的动态路由
```go
package main

import (
	"fmt"
	"net/http"
	"gee"
)

func main() {
	r := gee.New()
	r.GET("/", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})

	r.GET("/hello", func(w http.ResponseWriter, req *http.Request) {
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	})

	r.Run(":9999")
}
```
### gee/gee.go
创建文件夹`gee`，新建文件`gee.go`
```go
type HandlerFunc func(http.ResponseWriter, *http.Request)
```
定义了类型`HandlerFunc`:提供给使用框架的用户，用来定义路由映射的处理方法。
```go
type Engine struct {
	router map[string]HandlerFunc
}
``` 
在`Engine`中添加了一张路由映射表`router`
```go
func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	key := method + "-" + pattern
	engine.router[key] = handler
}
``` 
key 由请求方法`methond`和静态路由地址`pattern`构成，如`GET-/`、`POST-/hello`。这样针对相同的路由，如果请求方法不同，可以映射不同的处理方法(Handler)，value 是用户映射的处理方法
```go
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}
``` 
当用户调用`(*Engine).GET()`方法时，会将路由和处理方法注册到映射表 *router* 中
```go
func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}
```
`(*Engine).Run()`方法，是 *ListenAndServe* 的包装
```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	key := req.Method + "-" + req.URL.Path
	if handler, ok := engine.router[key]; ok {		// 查到了
		handler(w, req)
	} else {										// 没查到
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}
``` 
`Engine`实现的 *ServeHTTP* 方法的作用：解析请求的路径，查找路由映射表，如果查到就执行注册的处理方法。如果查不到，就返回 *404 NOT FOUND*
# 上下文 Context
- 将`路由(router)`独立出来，方便之后增强。
- 设计`上下文(Context)`，封装 Request 和 Response ，提供对 JSON、HTML 等返回类型的支持。
## 设计Context
### 必要性
1. 对Web服务来说，无非是根据请求`*http.Request`，构造响应`http.ResponseWriter`。但是这两个对象提供的接口粒度太细，比如要构造一个完整的响应，需要考虑消息头(Header)和消息体(Body)，Header 包含了状态码(StatusCode)，消息类型(ContentType)等几乎每次请求都需要设置的信息，如果不进行有效的封装，那么框架的用户将需要写大量重复，繁杂的代码，而且容易出错
2. 针对使用场景，封装`*http.Request`和`http.ResponseWriter`的方法，简化相关接口的调用，只是设计 Context 的原因之一。对于框架来说，还需要支撑额外的功能。例如，将来解析动态路由`/hello/:name`，参数`:name`的值放在哪呢？再比如，框架需要支持中间件，那中间件产生的信息放在哪呢？Context 随着每一个请求的出现而产生，请求的结束而销毁，和当前请求强相关的信息都应由 Context 承载。因此，设计 Context 结构，扩展性和复杂性留在了内部，而对外简化了接口。路由的处理函数，以及将要实现的中间件，参数都统一使用 Context 实例， Context 就像一次会话的百宝箱，可以找到任何东西。
## 具体实现
### context.go
```go
type H map[string]interface{}
``` 
给`map[string]interface{}`起了一个别名`gee.H`，构建JSON数据时显得更简洁
```go
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	// response info
	StatusCode int
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Writer: w,
		Req:    req,
		Path:   req.URL.Path,
		Method: req.Method,
	}
}
``` 
`Context`目前只包含了`http.ResponseWriter`和`*http.Request`，另外提供了对 Method 和 Path 这两个常用属性的直接访问。
```go
func (c *Context) PostForm(key string) string {
	return c.Req.FormValue(key)
}

func (c *Context) Query(key string) string {
	return c.Req.URL.Query().Get(key)
}
``` 
提供了访问 Query 和 PostForm 参数的方法
```go
func (c *Context) String(code int, format string, values ...interface{}) {
	c.SetHeader("Content-Type", "text/plain")
	c.Status(code)
	c.Writer.Write([]byte(fmt.Sprintf(format, values...)))
}

func (c *Context) JSON(code int, obj interface{}) {
	c.SetHeader("Content-Type", "application/json")
	c.Status(code)
	encoder := json.NewEncoder(c.Writer)
	if err := encoder.Encode(obj); err != nil {
		http.Error(c.Writer, err.Error(), 500)
	}
}

func (c *Context) Data(code int, data []byte) {
	c.Status(code)
	c.Writer.Write(data)
}

func (c *Context) HTML(code int, html string) {
	c.SetHeader("Content-Type", "text/html")
	c.Status(code)
	c.Writer.Write([]byte(html))
}
``` 
提供了快速构造 String/Data/JSON/HTML 响应的方法
### 路由(Router)
一开始，路由是写在`gee.go`里面的，现在将和路由相关的方法和结构提取出来，放到一个新的文件`router.go`中，方便下一次对 router 的功能进行增强，例如提供动态路由的支持。 
```go
func (r *router) handle(c *Context) {
	key := c.Method + "-" + c.Path
	if handler, ok := r.handlers[key]; ok {
		handler(c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}
``` 
router 的 handler 方法作了一个细微的调整，即 handler 的参数变成了 Context
### 框架入口
#### gee.go
将`router`相关的代码独立后，`gee.go`简单了不少。最重要的还是实现了 ServeHTTP 接口，接管了所有的 HTTP 请求。
```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := newContext(w, req)
	engine.router.handle(c)
}
``` 
相比之前的代码，这个方法也有细微的调整，在调用 `router.handle` 之前，构造了一个 Context 对象。这个对象目前还非常简单，仅仅是包装了原来的两个参数。
#### main.go
```go
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")		// HTML函数
	})
	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)	// String函数
	})

	r.POST("/login", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{				// JSON函数
			"username": c.PostForm("username"),
			"password": c.PostForm("password"),
		})
	})

	r.Run(":9999")
}
``` 
把`Handler`的参数变成成了`gee.Context`，提供了查询Query/PostForm参数的功能。
`gee.Context`封装了`HTML/String/JSON`函数，能够快速构造HTTP响应。
#### test
运行`main.go`，在`cmd`中使用`curl`查看成果：
```go
> curl -i http://localhost:9999/
HTTP/1.1 200 OK
Date: Mon, 12 Aug 2019 16:52:52 GMT
Content-Length: 18
Content-Type: text/html; charset=utf-8
<h1>Hello Gee</h1>

> curl "http://localhost:9999/hello?name=geektutu"
hello geektutu, you're at /hello

> curl "http://localhost:9999/login" -X POST -d "username=geektutu&password=1234"
{"password":"1234","username":"geektutu"}

> curl "http://localhost:9999/xxx"
404 NOT FOUND: /xxx
```
# 前缀树路由
使用 Trie 树实现动态路由(dynamic route)解析
支持两种模式`:name`和`*filepath`
```go

``` 
```go

``` 
```go

``` 

```go

``` 
```go

``` 
```go

``` 
