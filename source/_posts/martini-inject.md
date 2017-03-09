---
title: Martini路由框架
date: 2017-03-09 11:27:49
tags:
 - martini
 - golang
categories:
 - 后端
---

**[go-martini/martini](https://github.com/go-martini/martini)**是一个非常简单，流行的Golang web框架，并且非常的小巧简单灵活易扩展，也有非常多的插件，见**<https://github.com/martini-contrib>**

``` go
package main

import "github.com/go-martini/martini"

func main() {
    m := martini.Classic()
    m.Get("/", func() string {
    	return "Hello world!"
    })
    m.Run()
}
```

# Inject
**[go-martini/martini](https://github.com/go-martini/martini)**的灵活性多亏了**[codegangsta/inject](https://github.com/codegangsta/inject)**，后者是一个Golang的注入小框架，代码非常简单，利用Golang自带的reflect库完成函数参数的自动适配。

``` go
package main

import (
    "fmt"
    "github.com/codegangsta/inject"
)

// 自定义一个空interface{}, 必须是interface{}类型MapTo才能接受
type SpecialString interface{}

// 原定义Foo(s1,s2 string), 改成
func Foo(s1 string, s2 SpecialString) {
    fmt.Println(s1, s2.(string)) // type assertion
}

func main() {
    ij := inject.New()
    // 注入string类型的"a"
    ij.Map("a")
    // 注入SpecialString类型的"b"
    ij.MapTo("b", (*SpecialString)(nil))
    // 调用Foo函数时，会自动查找已经注入的值，并自动传递。
    // inject包的设计确保不会有两个相同类型的值（限制）
    // 若找不到会抛出运行时错误
    ij.Invoke(Foo)
}
```

Map和MapTo的区别在于后者可以指定注入value的类型

除了自动注入函数的参数外，还支持自动注入Struct的成员变量，参见Apply函数（利用了Struct成员的tag特性）

## (*SpecialString)(nil))
这种写法对于初学者很费解，原理得从interface{}的内部表述说起

为了支持reflect，interface{}内部存储了实际对象的Kind（类型）和Value（值），例如var v interface{} = "aaaa"，那么v的类型就是string，值就是"aaaa"，回到inject，由于Mapto第二个参数只需要一个类型，所以我们只需要初始化一个正确的Kind，Value为nil的interface{}即可。

由于Go是有区分Struct指针和对象，初始化一个对象需要分配额外的内存，而指针就很简单（学C的很好理解），所以这里起始就是传入的一个指针，并且value是nil，用最小的开销实现了一个类型的表述和传导。

``` go
type Str struct{
	b int
}

func main() {
	var b *Str = nil
	var a = (*Str)(nil)
    // 变量a,b都是一个类型为Str的空指针
	fmt.Print(a)
	fmt.Print(b)
}
```

而inject在内部会自动获取指针指向变量的类型，见
``` go
// InterfaceOf dereferences a pointer to an Interface type.
// It panics if value is not an pointer to an interface.
func InterfaceOf(value interface{}) reflect.Type {
	t := reflect.TypeOf(value)

	// 获取实际的类型，用for循环是为了确保多级指针（指向指针的指针）
	for t.Kind() == reflect.Ptr {
		t = t.Elem()
	}

	if t.Kind() != reflect.Interface {
		panic("Called ... interface. (*MyInterface)(nil)")
	}

	return t
}
```

# Martini
主要的逻辑参考【github.com/go-martini/martini/martini.go】和【github.com/go-martini/martini/router.go】

## 初始化
ClassicMartini用两个匿名对象实现继承

``` go
type ClassicMartini struct {
	*Martini
	Router
}

type Martini struct {
	inject.Injector
	handlers []Handler
	action   Handler
	logger   *log.Logger
}

func Classic() *ClassicMartini {
	r := NewRouter()
	m := New()
	m.Use(Logger())
	m.Use(Recovery())
	m.Use(Static("public"))
    // Mapto注入router处理器
	m.MapTo(r, (*Routes)(nil))
	m.Action(r.Handle)
	return &ClassicMartini{m, r}
}
```

m.Use用于注入中间件，存储在Martini.handlers，m.Action存储实际的路由处理入口，存储在Martini.action，他们的调用顺序见
``` go
func (c *context) handler() Handler {
	if c.index < len(c.handlers) {
		return c.handlers[c.index]
	}
    // 最后调用
	if c.index == len(c.handlers) {
		return c.action
	}
	panic("invalid index for context handler")
}
```

## 运行入口
``` go
// Run the http server on a given host and port.
func (m *Martini) RunOnAddr(addr string) {
	logger := m.Injector.Get(reflect.TypeOf(m.logger)).Interface().(*log.Logger)
	logger.Printf("listening on %s (%s)\n", addr, Env)
	logger.Fatalln(http.ListenAndServe(addr, m))
}
```

可以看到传入的m变量，它必须实现接口http.Handler接口，回调如下
``` go
func (m *Martini) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	m.createContext(res, req).run()
}

type context struct {
	inject.Injector
	handlers []Handler
	action   Handler
	rw       ResponseWriter
	index    int
}

func (m *Martini) createContext(res http.ResponseWriter, req *http.Request) *context {
	c := &context{inject.New(), m.handlers, m.action, NewResponseWriter(res), 0}
    // 从m继承注入的变量，比如logger之类的全局变量
	c.SetParent(m)
    // 注入自己
	c.MapTo(c, (*Context)(nil))
    // 注入responce
	c.MapTo(c.rw, (*http.ResponseWriter)(nil))
    // 注入request
	c.Map(req)
	return c
}

// 最后运行Run函数，先依次运行middleware，然后运行路由总入口，参见handler()函数
func (c *context) run() {
	for c.index <= len(c.handlers) {
        // 最终会触发router.Handle
		_, err := c.Invoke(c.handler())
		if err != nil {
			panic(err)
		}
		c.index += 1

		if c.Written() {
			return
		}
	}
}
```

在router.Handle找到最匹配的route，调用route.Handle函数

``` go
type router struct {
	// 所有的路由对象
	routes     []*route
	notFounds  []Handler
	groups     []group
	routesLock sync.RWMutex
}

```


``` go

type route struct {
	method   string
	regex    *regexp.Regexp
    // 所有的handle
	handlers []Handler
	pattern  string
	name     string
}

func (r *route) Handle(c Context, res http.ResponseWriter) {
	// 构建新的对象，
	context := &routeContext{c, 0, r.handlers}
    // 注入新的变量
	c.MapTo(context, (*Context)(nil))
	c.MapTo(r, (*Route)(nil))
    // 注意这里调用的是routeContext的run
	context.run()
}

type routeContext struct {
	// 继承了最初的Context，后者又继承了Martini注入的变量
	Context
	index    int
	handlers []Handler
}

// 依次触发所有的handle
func (r *routeContext) run() {
	for r.index < len(r.handlers) {
		handler := r.handlers[r.index]
        // 依次触发回调
		vals, err := r.Invoke(handler)
		if err != nil {
			panic(err)
		}
		r.index += 1

		// if the handler returned something, write it to the http response
		if len(vals) > 0 {
			ev := r.Get(reflect.TypeOf(ReturnHandler(nil)))
			handleReturn := ev.Interface().(ReturnHandler)
			handleReturn(r, vals)
		}

		if r.Written() {
			return
		}
	}
}
```

## 参考
1. <https://my.oschina.net/achun/blog/192912>
2. <http://betazk.github.io/2014/11/%E5%AF%B9martini%E8%BF%99%E4%B8%AA%E6%A1%86%E6%9E%B6%E4%B8%ADinject%E9%83%A8%E5%88%86%E7%9A%84%E7%90%86%E8%A7%A3/>
3. <http://memoryleak.io/go/martini/2015/10/24/martini-inject.html>