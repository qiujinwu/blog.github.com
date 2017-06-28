---
title: Golang Http调试
date: 2017-6-28 19:13:21
tags:
 - golang
categories:
 - Golang
---

Go Http包的处理函数如下，一个用于输入(Request)，一个用于输出（Response)，很多第三方的web框架都提供了自己的处理函数定义，但都很方便地适配HandlerFunc
``` go
type HandlerFunc func(ResponseWriter, *Request)
```

为了调试，http包提供了ResponseWriter/Request的模拟

# httptest
1. httptest.NewRequest 用于模拟一个Request请求
2. httptest.NewRecorder用于模拟一个Response

``` go
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"net/http/httptest"
)

func main() {
	handler := func(w http.ResponseWriter, r *http.Request) {
		io.WriteString(w, "<html><body>Hello World![" + r.URL.String() + "]</body></html>")
	}

	req := httptest.NewRequest("GET", "http://example.com/foo", nil)
	w := httptest.NewRecorder()
	handler(w, req)

	resp := w.Result()
	body, _ := ioutil.ReadAll(resp.Body)

	fmt.Println(resp.StatusCode)
	fmt.Println(resp.Header.Get("Content-Type"))
	fmt.Println(string(body))

	// Output:
	// 200
	// text/html; charset=utf-8
	// <html><body>Hello World![http://example.com/foo]</body></html>
}
```

# Server
处理模拟请求和响应，Httptest同样提供了服务器的模拟，不过非常简单，没有路由，只能提供单个响应函数（*在响应函数自行处理另当别论*）
``` go
import (
	"fmt"
	"io/ioutil"
	"net/http"
	"net/http/httptest"
	"log"
)

func main() {
	ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello, client")
	}))
	defer ts.Close()

	res, err := http.Get(ts.URL)
	if err != nil {
		log.Fatal(err)
	}
	greeting, err := ioutil.ReadAll(res.Body)
	res.Body.Close()
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("%s", greeting)
	// Output: Hello, client
}
```

server提供了几个接口，用法有一些简单的区别
``` go
// NewServer starts and returns a new Server.
// The caller should call Close when finished, to shut it down.
func NewServer(handler http.Handler) *Server

// NewUnstartedServer returns a new Server but doesn't start it.
//
// After changing its configuration, the caller should call Start or
// StartTLS.
//
// The caller should call Close when finished, to shut it down.
func NewUnstartedServer(handler http.Handler) *Server

// NewTLSServer starts and returns a new Server using TLS.
// The caller should call Close when finished, to shut it down.
func NewTLSServer(handler http.Handler) *Server
```

启动服务器，并不会阻塞当前gorounte，在内部会开启一个新的gorounte来执行监听任务
``` go
func (s *Server) goServe() {
	s.wg.Add(1)
	go func() {
		defer s.wg.Done()
		s.Config.Serve(s.Listener)
	}()
}
```

# httptrace
此外，http还包含httptrace，用于监听http请求的各种事件，核心就是一个ClientTrace对象，具体的函数，可以godoc
``` go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"net/http/httptrace"
	"os"
)

func main() {
	server := httptest.NewServer(http.HandlerFunc(http.NotFound))
	defer server.Close()
	c := http.Client{}
	req, err := http.NewRequest("GET", server.URL, nil)
	if err != nil {
		panic(err)
	}

	trace := &httptrace.ClientTrace{
		GotConn: func(connInfo httptrace.GotConnInfo) {
			fmt.Println("Got Conn")
		},
		ConnectStart: func(network, addr string) {
			fmt.Println("Dial start")
		},
		ConnectDone: func(network, addr string, err error) {
			fmt.Println("Dial done")
		},
		GotFirstResponseByte: func() {
			fmt.Println("First response byte!")
		},
		WroteHeaders: func() {
			fmt.Println("Wrote headers")
		},
		WroteRequest: func(wr httptrace.WroteRequestInfo) {
			fmt.Println("Wrote request", wr)
		},
	}
	req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
	fmt.Println("Starting request!")
	resp, err := c.Do(req)
	if err != nil {
		panic(err)
	}
	io.Copy(os.Stdout, resp.Body)
	fmt.Println("Done!")
}

```

# 参考
1. <https://golang.org/src/net/http/httptest/example_test.go>
1. <https://golang.org/pkg/net/http/httptrace/>
2. <http://www.tuicool.com/articles/nQzqmuZ>