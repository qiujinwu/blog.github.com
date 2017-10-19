---
title: Kelly源码剖析
date: 2017-10-18 20:36:15
tags:
 - golang
categories:
 - Golang
---

[Kelly](https://github.com/qjw/kelly)是基于golang的一个简单的web框架

# 背景

作为web后端开发，标准的[net/http](https://golang.org/pkg/net/http/)非常高效灵活，足以适用非常多的场景，当然也有很多周边待补充，这就出现了各种web框架，甚至出现了替代默认的Http库的[valyala/fasthttp](https://github.com/valyala/fasthttp)

golang目前百花齐放，个人主要了解到的是两个项目

1. [beego: simple & powerful Go app framework](https://beego.me/)
1. [gin-gonic/gin](https://github.com/gin-gonic/gin)

beego没有实际用过，听说是大而全的项目，对开发者友好。不过由于了解甚少，草率评论并不合适，这里不作过多说明。

本着刨根问底的学习态度，最开始了解的是[martini](https://github.com/philsong/martini)，后查证效率偏低（大量用到[反射/reflect](https://golang.org/pkg/reflect/)），所以就进一步学习了[gin-gonic/gin](https://github.com/gin-gonic/gin)。

后者小巧灵活，学习成本低，并且提供了很多实用的补充，例如

1. 路由和中间件核心框架，路由基于[julienschmidt/httprouter](https://github.com/julienschmidt/httprouter)
1. gin.Context
1. binding
1. 校验，基于[go-playground/validator.v9](https://gopkg.in/go-playground/validator.v9)
1. Http Request工具函数，获取param/path/form/header/cookie等
1. Http Response工具函数，设置cookie，header，返回xml/json，返回template支持等
1. 内建的几个常用中间件

martini/gin都包含非常多的中间件，两者迁移非常容易，参考

1. <https://github.com/codegangsta/martini-contrib>
1. <https://github.com/gin-gonic/contrib>
1. <https://github.com/gin-contrib>

用久了，也发现gin也有一些问题

1. 依赖还是偏多（*虽然和很多库相比算较少的*），就写个hello world都下载半天依赖
1. 第三方middleware有的依赖gopkg.in的代码，另外一些依赖github.com的代码
1. gin.Context对Golang标准库[context](https://golang.org/pkg/context/)不友好
1. binding有一些问题，本人的优化版本在<https://github.com/qjw/go-gin-binding>
1. 虽然middleware很多，但选择性太多，质量参差不齐，不好选择，另外太多的第三方依赖不如将大部分常用的集成到一起来的方便。

经过多方对比考察，认为[urfave/negroni](https://github.com/urfave/negroni)作为路由/中间件基础框架非常合适，（*看看原型就知道他对[context](https://golang.org/pkg/context/)有多友好*）所以折腾就开始了。

``` golang
type Handler interface {
  ServeHTTP(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc)
}
```

经过综合评估，决定自己弄个类似于gin的框架

原则是尽量踏着巨人的肩膀，避免一些通用组件重复造轮子，聚焦于优秀智慧的集成


# 安装测试
安装kelly
``` bash
go get github.com/qjw/kelly
```

运行sample
``` bash
go get github.com/qjw/kelly/sample
```

具体参考<https://github.com/qjw/kelly#运行sample>

``` bash
# 运行sample
king@king:~/tmp/gopath/bin$ ./sample
[negroni] listening on :9090
```

## 源码结构
1. .(当前目录)：核心代码
1. binding：数据绑定支持，必需，自动安装
1. render：响应输出支持，必须，自动安装，例如响应json/xml/html/text/模板/二进制，以及重定向等
1. sample：测试代码
1. sample-conf：测试代码
1. toolkits：可选的辅助工具集，例如二维码/验证码/模板引擎，
1. sessions：session/flash/认证/权限控制，可选，依赖redis
1. middleware：各种中间件，可选
1. middleware/swagger：swagger支持

# 路由
使用<https://github.com/julienschmidt/httprouter>

router需要支持以下特性
1. 各种http方法
2. Path变量
3. 多级路由

httprouter并不原生多级路由，所以这里做了些扩展【**留意代码中的注释，下同**】

``` go
type Router interface {
	// 支持的http方法，支持链式调用
	GET(string, ...HandlerFunc) Router
	HEAD(string, ...HandlerFunc) Router
	OPTIONS(string, ...HandlerFunc) Router
	POST(string, ...HandlerFunc) Router
	PUT(string, ...HandlerFunc) Router
	PATCH(string, ...HandlerFunc) Router
	DELETE(string, ...HandlerFunc) Router
}
type router struct {	
	// 共享的全局httprouter
	rt *httprouter.Router
	// 当前rouer路径
	path string
	// 绝对路径
	absolutePath string
}

func (rt *router) GET(path string, handles ...HandlerFunc) Router {
	// 留意rt.rt.GET作为参数传入
	return rt.methodImp(rt.rt.GET, "GET", path, handles...)
}

func (rt *router) methodImp(
	handle func(path string, handle httprouter.Handle),
	method string,
	path string,
	handles ...HandlerFunc) Router {

	// 增加计数
	rt.endpoints = append(rt.endpoints, &endpoint{
		method:  method,
		path:    path,
		handles: handles,
		endPointRegisterCB: func() {
			// 在调用传入的handle函数前，已经加上了rt.absolutePath
			handle(rt.absolutePath+path, rt.wrapHandle(handles...))
		},
	})
	return rt
}
```

kelly的router都共享一个全局的httprouter对象，并且各自保存自己的所处的路径（path），在绑定特定的http请求时，自动加上自己的前缀，再绑定到原生的httprouter上去

# Callback和Context

golang自带的net/http callback为
``` go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

Context则是一个输入（Request），一个输出/响应（ResponseWriter），这种方法有很好的灵活性，不过易用性稍弱

kelly借鉴gin的做法
``` go
type HandlerFunc func(c *Context)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	f(newContext(w, r, nil))
}
```

其中最关键的就是这个Context，因为web编程大部分工作（刨去业务代码），最重要的还是解析输入，构建输出，这些都是基于http请求，这就是Context的抽象
``` go
type Context struct {
	http.ResponseWriter
	r *http.Request

	// 下一个处理逻辑，用于middleware
	next http.HandlerFunc

	// 用于支持设置context数据
	dataContext
	// render
	renderOp
	// request
	request
	// binder
	binder
}
```

可以看到Context除了包装了http.Request和http.ResponseWriter之外，还有一些
1. dataContext ：支持请求context，用于在中间件链传递数据。**参见源码目录render**
2. renderOp ：格式化输出支持（例如xml/json等）
3. binder ： 数据绑定支持，**参见源码目录binding**
4. request ：输入（获取参数）支持

binder和request在于，前者是自动将http请求的输入绑定到一个golang struct，并且依据规则进行校验，提高开发效率。后者则属于一些工具函数，例如获取某个param/path/form参数，或者获取某个cookie。

``` go
type binder interface {
	// 绑定一个对象，根据Content-type自动判断类型
	Bind(interface{}) (error, []string)
	// 绑定json，从body取数据
	BindJson(interface{}) (error, []string)
	// 绑定xml，从body取数据
	BindXml(interface{}) (error, []string)
	// 绑定form，从body/query取数据
	BindForm(interface{}) (error, []string)
	// 绑定path变量
	BindPath(interface{}) (error, []string)

	GetBindParameter() interface{}
	GetBindJsonParameter() interface{}
	GetBindXmlParameter() interface{}
	GetBindFormParameter() interface{}
	GetBindPathParameter() interface{}
}
```
``` go
type request interface {
	// 根据key获取cookie值
	GetCookie(string) (string, error)
	// 根据key获取cookie值，若不存在，则返回默认值
	GetDefaultCookie(string, string) string
	// 根据key获取cookie值，若不存在，则panic
	MustGetCookie(string) string

	// 根据key获取header值
	GetHeader(string) (string, error)
	// 根据key获取header值，若不存在，则返回默认值
	GetDefaultHeader(string, string) string
	// 根据key获取header值，若不存在，则panic
	MustGetHeader(string) string
	// Content-Type
	ContentType() string

	// 根据key获取PATH变量值
	GetPathVarible(string) (string, error)
	// 根据key获取PATH变量值，若不存在，则panic
	MustGetPathVarible(string) string

	// 根据key获取QUERY变量值，可能包含多个（http://127.0.0.1:9090/path/abc?abc=bbb&abc=aaa）
	GetMultiQueryVarible(string) ([]string, error)
	// 根据key获取QUERY变量值，仅返回第一个
	GetQueryVarible(string) (string, error)
	// 根据key获取QUERY变量值，仅返回第一个,若不存在，则返回默认值
	GetDefaultQueryVarible(string, string) string
	// 根据key获取QUERY变量值，仅返回第一个,若不存在，则panic
	MustGetQueryVarible(string) string

	// 根据key获取FORM变量值，可能get可能包含多个
	GetMultiFormVarible(string) ([]string, error)
	// 根据key获取FORM变量值，仅返回第一个
	GetFormVarible(string) (string, error)
	// 根据key获取FORM变量值，仅返回第一个,若不存在，则返回默认值
	GetDefaultFormVarible(string, string) string
	// 根据key获取FORM变量值，仅返回第一个,若不存在，则panic
	MustGetFormVarible(string) string

	// @ref http.Request.ParseMultipartForm
	ParseMultipartForm() error
	// 获取（上传的）文件信息
	GetFileVarible(string) (multipart.File, *multipart.FileHeader, error)
	MustGetFileVarible(string) (multipart.File, *multipart.FileHeader)
}
```

render处理常见的xml/json等之外，还有比如设置cookie/header等操作，完成的功能列表如下
``` go
type renderOp interface {
	// 返回紧凑的json
	WriteJson(int, interface{})
	// 返回xml
	WriteXml(int, interface{})
	// 返回html
	WriteHtml(int, string)
	// 返回模板html
	WriteTemplateHtml(int, *template.Template, interface{})
	// 返回格式化的json
	WriteIndentedJson(int, interface{})
	// 返回文本
	WriteString(int, string, ...interface{})
	// 返回二进制数据
	WriteData(int, string, []byte)
	// 返回重定向
	Redirect(int, string)
	// 设置header
	SetHeader(string, string)
	// 设置cookie
	SetCookie(string, string, int, string, string, bool, bool)

	Abort(int, string)
	ResponseStatusOK()
	ResponseStatusBadRequest(error)
	ResponseStatusUnauthorized(error)
	ResponseStatusForbidden(error)
	ResponseStatusNotFound(error)
	ResponseStatusInternalServerError(error)
}
```

**这里的WriteTemplateHtml使用golang内置的模板引擎，由于功能有限，在【toolkits/template】中基于第三方引擎实现更为强大的功能**

context则比较简单，只是一个数据的get/set
``` go
type dataContext interface {
	Set(interface{}, interface{}) dataContext
	Get(interface{}) interface{}
	MustGet(interface{}) interface{}
}
```

# 中间件框架
好的中间件框架是一个web框架灵活性的重要标志，目前主流的框架大部分都支持中间件扩展，并且有丰富的第三方中间件。为了方便移植其他框架的中间件，一般都容易兼容标准的net/http接口。

标准的net/http默认不支持读写自定义数据，这很大程度影响中间件的灵活性，当然可以用标准库[context](https://golang.org/pkg/context/)进行扩充，由于需要替换原有的reqeust，对程序结构影响很大，参见<http://www.flysnow.org/2017/07/29/go-classic-libs-gorilla-context.html#新的替代者>。经过调研，选用<https://github.com/urfave/negroni>

negroni原型如下
``` go
type HandlerFunc func(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc)

func (h HandlerFunc) ServeHTTP(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
	h(rw, r, next)
}
```
**negroni中间件默认不会传递调用，而是需要手动触发调用参数next，所以当使用context添加data并替换了http.Request对象时，处理起来就很轻松和自然**。

## 回调调用链

kelly的入口在kelly.Run
``` go

type kellyImp struct {
	*router
	n *negroni.Negroni
}

func (k *kellyImp) Run(addr ...string) {
	// 运行negroni.Negroni.Run
	k.n.Run(addr...)
}

func newImp(n *negroni.Negroni, handlers ...HandlerFunc) Kelly {
	// 创建negroni.Negroni回调
	rt := newRouterImp(handlers...)
	ky := &kellyImp{
		router: rt,
		n:      n,
	}
	ky.n = n
	
	// negroni触发之后，会进入rt
	n.UseHandler(rt)
	return ky
}
```

当kelly收到http请求，先触发negroni.Negroni的回调，这个回调在router定义

``` go
func newRouterImp(handlers ...HandlerFunc) *router {
	// 创建全局的httprouter对象
	httpRt := httprouter.New()
	// 创建根router对象
	rt := &router{
		rt:           httpRt,
		path:         "", // 根router路径是空的
		absolutePath: "",
		dataContext:  newMapContext(),
	}
	return rt
}
```

而router又将请求转发到了全局的httprouter对象，参见下面代码


``` go
func (rt *router) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
	rt.rt.ServeHTTP(rw, r)
}
```

最终就触发了我们在注册http方法时设置到httprouter的回调中

## 中间件
1. 有两种方式设置中间件，在创建router时，传入
2. 创建之后，调用Use

``` go
// 创建根router
func NewClassic(handlers ...HandlerFunc) Kelly {
	return newImp(negroni.Classic(), handlers...)
}

// 创建根router
func New(handlers ...HandlerFunc) Kelly {
	return newImp(negroni.New(negroni.NewRecovery()), handlers...)
}

type Router interface {
	// 新建子router
	Group(string, ...HandlerFunc) Router

	// 动态插入中间件
	Use(...HandlerFunc) Router
}
```

由于httprouter并没有对中间件的支持，所以需要继续作转换

最外层的注册，使用了kelly的回调原型
``` go
type router struct {
	// 中间件
	middlewares []HandlerFunc

	// 所有的子Group
	groups []*router

	// 父Group
	parent *router

	endpoints []*endpoint
}

func (rt *router) GET(path string, handles ...HandlerFunc) Router {
	return rt.methodImp(rt.rt.GET, "GET", path, handles...)
}
```

留意**结构中的endPointRegisterCB成员**和**rt.wrapHandle(handles...)**

``` go
func (rt *router) methodImp(
	handle func(path string, handle httprouter.Handle),
	method string,
	path string,
	handles ...HandlerFunc) Router {

	// 将注册信息加入到endpoints成员中，
	rt.endpoints = append(rt.endpoints, &endpoint{
		method:  method,
		path:    path,
		handles: handles,
		endPointRegisterCB: func() {
			// 注册回调，
			handle(rt.absolutePath+path, rt.wrapHandle(handles...))
		},
	})
	return rt
}
```
在kelly进入主循环之前，会完成调用。**留意doBeforeRun**
``` go
func (k *kellyImp) Run(addr ...string) {
	if k.n == nil {
		panic("invalid kelly")
	}
	k.router.doBeforeRun()
	k.n.Run(addr...)
}
```
``` go
func (rt *router) doBeforeRun() {
	// 先注册自己的endpoints
	for _, v := range rt.endpoints {
		v.run()
	}
	// 先注册儿子们的endpoints
	for _, v := range rt.groups {
		v.doBeforeRun()
	}
}
```
``` go
func (this *endpoint) run() {
	if DebugFlag && this.endPointRegisterCB == nil {
		panic("invalid endpoint")
	}
	// 直接调用创建时的函数，并且清空，避免重复调用
	this.endPointRegisterCB()
	this.endPointRegisterCB = nil
}
```

**之所以通过这种方式，在最后一步统一注册，是为了支持创建router之后使用Use方法动态添加中间件的情形**

## Callback转换

在endPointRegisterCB中，注册的回调需要原型【httprouter.Handle】，而我们传入的是【kelly.HandlerFunc数组】，通过下面的函数转换
``` go
func (rt *router) wrapHandle(handles ...HandlerFunc) httprouter.Handle {
	// 创建一个negroni实例
	tmpHandle := negroni.New()
	// 将父router的中间件（【kelly.HandlerFunc数组】）注册到negroni
	rt.wrapParentHandle(tmpHandle)

	// 注册当前router的中间件到negroni
	for _, v := range rt.middlewares {
		tmpHandle.UseFunc(wrapHandlerFunc(v))
	}

	// 注册特定方法的中间件到negroni
	for _, v := range handles {
		tmpHandle.UseFunc(wrapHandlerFunc(v))
	}

	// 返回一个httprouter的回调
	return func(wr http.ResponseWriter, r *http.Request, params httprouter.Params) {
		// 将请求转到negroni
		tmpHandle.ServeHTTP(wr, r)
	}
}

func (rt *router) wrapParentHandle(n *negroni.Negroni) {
	if rt.parent != nil {
		// 让父router优先注册自己的中间件
		rt.parent.wrapParentHandle(n)
		// 注册parent的中间件
		for _, v := range rt.parent.middlewares {
			n.UseFunc(wrapHandlerFunc(v))
		}
	}
}
```

大体的思路就是每次注册http方法，就生成一个negroni对象，并且注册祖宗十八代的中间件、自己的中间件和自己的处理函数，然后转成httprouter的回调注册。

而negroni包装kelly.HandlerFunc回调时，同样需要一层转换
``` go
func wrapHandlerFunc(f HandlerFunc) negroni.HandlerFunc {
	return func(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
		f(newContext(rw, r, next))
	}
}
```

**留意这个next**，为了支持中间件链继续运行，需要保存这个next

``` go
func newContext(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) *Context {
	c := &Context{
		next:           next,
	}
	return c
}
```
``` go
func (c *Context) InvokeNext() {
	if c.next != nil {
		c.next.ServeHTTP(c, c.Request())
	} else {
		panic("invalid invoke next")
	}
}
```

## 性能
一个请求进来，先走negroni，转发到httprouter，httprouter根据路由规则找到对应的negroni回调，然后触发每一个中间件回调，最后到业务回调。这个过程中，需要将negroni参数转换成kelly.HandlerFunc

这整个过程中存在

1. 额外的内存消耗，每个请求都会有一个negroni实例
2. 函数调用栈过长带来的cpu消耗
3. 每个中间件/回调都伴随这一个kelly.Context对象的创建（创建本身比较简单）

# Path变量

path变量 httprouter支持，具体存储在回调的第三个参数中

``` go
type Handle func(http.ResponseWriter, *http.Request, Params)

type Param struct {
	Key   string
	Value string
}

type Params []Param
```

回到函数wrapHandle，**留意mapContextFilter**
``` go
func (rt *router) wrapHandle(handles ...HandlerFunc) httprouter.Handle {
	return func(wr http.ResponseWriter, r *http.Request, params httprouter.Params) {
		r = mapContextFilter(wr, r, params)
		tmpHandle.ServeHTTP(wr, r)
	}
}
```

实际上就是将这个params参数通过标准库[context](https://golang.org/pkg/context/)存入request对象中

``` go
type contextMap map[interface{}]interface{}
func mapContextFilter(_ http.ResponseWriter, r *http.Request, params httprouter.Params) *http.Request{
	contextMap := contextMap{
		pathParamID: params,
	}
	return contextSet(r, contextKey, contextMap)
}
func contextSet(r *http.Request, key, value interface{}) *http.Request {
	ctx := context.WithValue(r.Context(), key, value)
	return r.WithContext(ctx)
}
```

## 取变量
``` go
func contextMustGet(r *http.Request, key interface{}) interface{} {
	v := r.Context().Value(key)
	if v == nil {
		panic(fmt.Errorf("get context value fail by '%v'", key))
	}
	return v
}

func getPathParams(r *http.Request) httprouter.Params{
	datas := contextMustGet(r, contextKey).(contextMap)
	// 这个pathParamID是一个全局常量
	return datas[pathParamID].(httprouter.Params)
}
```
``` go
func (r requestImp) GetPathVarible(name string) (string, error) {
	params := getPathParams(r.Request)
	val := params.ByName(name)
	if len(val) > 0 {
		return val, nil
	} else {
		return val, fmt.Errorf("can not get path varibel by '%v'", name)
	}
}
```

# Context数据
由于kelly.Context在整个调用链并非连续(而是每个negroni调用动态创建的)，所以不能简单地在里面加一个map之类的成员

``` go
func newContext(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) *Context {
	c := &Context{
		dataContext:    newMapHttpContext(r),
	}
	return c
}
```

所以和path变量一样，同样将一个map对象绑定到request对象
``` go
type mapHttpContext struct {
	r *http.Request
}

func (c *mapHttpContext) Set(key, value interface{}) dataContext {
	datas := contextMustGet(c.r, contextKey).(contextMap)
	datas[key] = value
	return c
}

func (c mapHttpContext) Get(key interface{}) interface{} {
	datas := contextMustGet(c.r, contextKey).(contextMap)
	if data, ok := datas[key]; ok {
		return data
	} else {
		return nil
	}
}

func newMapHttpContext(r *http.Request) dataContext {
	c := &mapHttpContext{
		r: r,
	}
	return c
}
```

# 注解

中间件可以在每个请求之前做些处理，甚至拦截，**注解则在每次注册Http请求时做些处理**，比如swagger就依赖于这种场景

``` go
type Router interface {
	// 添加全局的 注解 函数。该router下面和子（孙）router下面的endpoint注册都会被触发
	GlobalAnnotation(handles ...AnnotationHandlerFunc) Router

	// 添加临时 注解 函数，只对使用返回的AnnotationRouter对象进行注册的endpoint有效
	Annotation(handles ...AnnotationHandlerFunc) AnnotationRouter

	// 用于支持设置context数据
	dataContext
}

type AnnotationHandlerFunc func(c *AnnotationContext)

type AnnotationContext struct {
	r       Router
	method  string
	path    string
	// 不包含中间件
	handles []HandlerFunc
}
```

## 注册filter
``` go
func (rt *router) GlobalAnnotation(handles ...AnnotationHandlerFunc) (r Router) {
	r = rt
	if len(rt.epMiddlewares) == 0 {
		rt.epMiddlewares = make([]AnnotationHandlerFunc, len(handles))
		copy(rt.epMiddlewares, handles)
	} else {
		for _, item := range handles {
			rt.epMiddlewares = append(rt.epMiddlewares, item)
		}
	}
	return
}
```
只是简单地将他存到成员epMiddlewares中

``` go
type router struct {
	// endpoint钩子函数
	epMiddlewares []AnnotationHandlerFunc

	// 被子类覆盖的方法，
	overiteInvokeAnnotation func(c *AnnotationContext)
}
```

## 触发

创建router时，会指定rt。在doBeforeRun时，会完成注入
``` go
func newRouterImp(handlers ...HandlerFunc) *router {
	rt := &router{}
	rt.overiteInvokeAnnotation = rt.invokeAnnotation
	return rt
}

func (rt *router) invokeParentAnnotation(c *AnnotationContext) {
	if rt.parent != nil {
		rt.parent.invokeParentAnnotation(c)
		for _, item := range rt.parent.epMiddlewares {
			item(c)
		}
	}
}

func (rt *router) invokeAnnotation(c *AnnotationContext) {
	rt.invokeParentAnnotation(c)

	// 执行全局的ep 过滤器
	for _, item := range rt.epMiddlewares {
		item(c)
	}
}
```

再回到注册http请求的函数methodImp。**留意函数变量f**
``` go
func (rt *router) methodImp(
	handle func(path string, handle httprouter.Handle),
	method string,
	path string,
	handles ...HandlerFunc) Router {

	f := rt.overiteInvokeAnnotation

	// 增加计数
	rt.endpoints = append(rt.endpoints, &endpoint{
		endPointRegisterCB: func() {
			// 注册到httprouter
			handle(rt.absolutePath+path, rt.wrapHandle(handles...))
			// 调用注解
			f(&AnnotationContext{
				r:       rt,
				method:  method,
				path:    path,
				handles: handles,
			})
		},
	})
	return rt
}
```

## 临时Filter

使用GlobalAnnotation注册filter会应用到当前router和他的子router，临时filter需要使用kelly.Router.Annotation。这个函数返回的是一个AnnotationRouter对象。

这个对象有自己的AnnotationHandlerFunc数组，所以不会影响其他的请求

``` go
func newAnnotationRouter(r *router, handles ...AnnotationHandlerFunc) AnnotationRouter {
	return &annotationRouter{
		router:      r,
		middlewares: handles,
	}
}

type annotationRouter struct {
	*router
	// endpoint钩子函数
	middlewares []AnnotationHandlerFunc
}
```

同时为了在触发filter时，一并触发自己的filter，需要重写
``` go
func (r *annotationRouter) doMethod(
	f func(path string, handles ...HandlerFunc) Router,
	path string,
	handles ...HandlerFunc,
) Router {
	old := r.router.overiteInvokeAnnotation
	// 重写overiteInvokeAnnotation，并且函数退出自动复原
	r.router.overiteInvokeAnnotation = r.invokeAnnotation
	defer func() {
		r.router.overiteInvokeAnnotation = old
	}()
	return f(path, handles...)
}
```

**由于go并非真正的继承，而只是简单的组合，所以这里的多态实现有些另类**

## 注解使用

必须使用链式调用，或者保存返回的临时对象
``` go
router := r.Group("/swagger"
).GlobalAnnotation(swagger.SetGlobalParam(&swagger.StructParam{
	Tags: []string{"API接口"},
})).OPTIONS("/*path", func(c *kelly.Context) {
	c.ResponseStatusOK()
})
```
``` go
router.Annotation(swagger.Swagger(&swagger.StructParam{
	ResponseData: &swagger.SuccessResp{},
	FormData:     &swaggerParam{},
	Summary:      "api1",
})).POST("/api1", func(c *kelly.Context) {
	c.ResponseStatusOK()
})
```


