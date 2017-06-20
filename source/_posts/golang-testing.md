---
title: Golang单元测试
date: 2017-6-20 12:23:16
tags:
 - golang
categories:
 - Golang
---

# 简单用法
记住下面的这些原则：

1. 文件名必须是_test.go结尾的，这样在执行go test的时候才会执行到相应的代码
1. 你必须import testing这个包
1. 所有的测试用例函数必须是Test开头
1. 测试用例会按照源代码中写的顺序依次执行
1. 测试函数TestXxx()的参数是testing.T，我们可以使用该类型来记录错误或者是测试状态
1. 测试格式：func TestXxx (t *testing.T),Xxx部分可以为任意的字母数字的组合，但是首字母不能是小写字母[a-z]，例如Testintdiv是错误的函数名。
1. .T的Error, Errorf, FailNow, Fatal, FatalIf方法，说明测试不通过，调用Log方法用来记录测试的信息。

例如 fibonacci.go
``` go
package lib

//斐波那契数列
//求出第n个数的值
func Fibonacci(n int64) int64 {
    if n < 2 {
        return n
    }
    return Fibonacci(n-1) + Fibonacci(n-2)
}
```
fibonacci_test.go
``` go
package lib

import (
    "testing"
)
 
func TestFibonacci(t *testing.T) {
    r := Fibonacci(10)
    if r != 55 {
        t.Errorf("Fibonacci(10) failed. Got %d, expected 55.", r)
    }
}
```
``` bash
go test lib
```

## 构造析构
``` go
// package level initialization of database connections
func init() {
	// init database connections
}

// 该函数由 go 语言的 test 框架调用
func TestLastUnit(t *testing.T) {
    // 测试结束时，清理数据
    defer func(userID string) {
    }(userID)
}
```

如果待测试的功能模块涉及到文件操作，临时文件是一个不错的解决方案,**go语言的 ioutil 包提供了 TempDir 和 TempFile 方法，供我们使用。**

## Package
在写单元测试时，一般情况下，我们将功能代码和测试代码放到同一个目录下，仅以后缀 _test 进行区分。

对于复杂的大型项目，功能依赖比较多时，通常在跟目录下再增加一个 test 文件夹，不同的测试放到不同的子目录下面，如下图所示：

## 覆盖率
在go语言的测试覆盖率统计时，go test通过参数**covermode**的设定可以对覆盖率统计模式作如下三种设定。
1. set	缺省模式, 只记录语句是否被执行过
1. count	记录语句被执行的次数
1. atomic	记录语句被执行的次数，并保证在并发执行时的正确性

其他选项
1.  -cover 允许代码分析
1. -coverprofile 输出结果文件

``` bash
go test -cover -coverprofile=cover.out -covermode=count -o /tmp/testgo test
go tool cover -func=cover.out
# 用html直观展示
go tool cover -html=cover.out
```

## httptest
针对模拟网络访问，标准库了提供了一个httptest包，可以让我们模拟http的网络调用，下面举个例子了解使用。
``` go
package test

import (
	"io"
	"net/http"
)

// e.g. http.HandleFunc("/health-check", HealthCheckHandler)
func HealthCheckHandler(w http.ResponseWriter, r *http.Request) {
	// A very simple health check.
	w.WriteHeader(http.StatusOK)
	w.Header().Set("Content-Type", "application/json")

	// In the future we could report back on the status of our DB, or our cache
	// (e.g. Redis) by performing a simple PING, and include them in the response.
	io.WriteString(w, `{"alive": true}`)
}
```
``` go
package test

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHealthCheckHandler(t *testing.T) {
	// Create a request to pass to our handler. We don't have any query parameters for now, so we'll
	// pass 'nil' as the third parameter.
	req, err := http.NewRequest("GET", "/health-check", nil)
	if err != nil {
		t.Fatal(err)
	}

	// We create a ResponseRecorder (which satisfies http.ResponseWriter) to record the response.
	rr := httptest.NewRecorder()
	handler := http.HandlerFunc(HealthCheckHandler)

	// Our handlers satisfy http.Handler, so we can call their ServeHTTP method
	// directly and pass in our Request and ResponseRecorder.
	handler.ServeHTTP(rr, req)

	// Check the status code is what we expect.
	if status := rr.Code; status != http.StatusOK {
		t.Errorf("handler returned wrong status code: got %v want %v",
			status, http.StatusOK)
	}

	// Check the response body is what we expect.
	expected := `{"alive1": true}`
	if rr.Body.String() != expected {
		t.Errorf("handler returned unexpected body: got %v want %v",
			rr.Body.String(), expected)
	}
}

```

# stretchr/testify

和其他的单元测试相比，go提供的默认方案灵活但繁琐，<https://github.com/stretchr/testify>作了适当的包装简化

``` bash
go get github.com/stretchr/testify
```

testify又分成几个模块

## assert
1. Prints friendly, easy to read failure descriptions
1. Allows for very readable code
1. Optionally annotate each assertion with a message

``` go
package yours

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestSomething(t *testing.T) {

    // assert equality
    assert.Equal(t, 123, 123, "they should be equal")

    // assert inequality
    assert.NotEqual(t, 123, 456, "they should not be equal")

    // assert for nil (good for errors)
    assert.Nil(t, object)

    // assert for not nil (good when you expect something)
    if assert.NotNil(t, object) {

    // now we know that object isn't nil, we are safe to make
    // further assertions without causing any errors
    assert.Equal(t, "Something", object.Value)

    }
}
```

## require
The require package provides same global functions as the assert package, but instead of returning a boolean result they terminate current test.

## mock
当某些接口依赖其他接口时，可以通过mock模拟依赖的接口并做出预期的输出
``` go
package test

type Random interface {
	Random(limit int) int
}

type Calculator interface {
	Random() int
}

func newCalculator(rnd Random) Calculator {
	return calc{
		rnd: rnd,
	}
}

type calc struct {
	rnd Random
}

func (c calc) Random() int {
	return c.rnd.Random(100)
}
```
Calculator 接口创建依赖于Random接口，为了测试前者，我们用mock**模拟一个Random接口**，见randomMock
``` go
package test

import (
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
	"testing"
)

type randomMock struct {
	mock.Mock
}

func (o randomMock) Random(limit int) int {
	args := o.Called(limit)
	return args.Int(0)
}

func TestRandom(t *testing.T) {
	rnd := new(randomMock)
	rnd.On("Random", 100).Return(7)
	calc := newCalculator(rnd)
	assert.Equal(t, 7, calc.Random())
}
```

### mockery
**[mockery](https://github.com/vektra/mockery)** provides the ability to easily generate mocks for golang interfaces. It removes the boilerplate coding required to use mocks.

接上面的例子，运行下面的命令
``` bash
go get github.com/vektra/mockery/.../
```
``` bash
king@king:~/code/go/src/test$ mockery -name=Random
Generating mock for: Random
```
就自动生成mocks/Random.go

```
├── main.go
├── main_test.go
├── mocks
│   └── Random.go
```
``` go
// Code generated by mockery v1.0.0
package mocks

import mock "github.com/stretchr/testify/mock"

// Random is an autogenerated mock type for the Random type
type Random struct {
	mock.Mock
}

// Random provides a mock function with given fields: limit
func (_m *Random) Random(limit int) int {
	ret := _m.Called(limit)

	var r0 int
	if rf, ok := ret.Get(0).(func(int) int); ok {
		r0 = rf(limit)
	} else {
		r0 = ret.Get(0).(int)
	}

	return r0
}
```

对应的main_test.go修改为
``` go
package test

import (
	"github.com/stretchr/testify/assert"
	// "github.com/stretchr/testify/mock"
	"testing"
	"test/mocks"
)

//type RandomMock struct {
//	mock.Mock
//}
//
//func (o RandomMock) Random(limit int) int {
//	args := o.Called(limit)
//	return args.Int(0)
//}

func TestRandom(t *testing.T) {
	rnd := new(mocks.Random)
	rnd.On("Random", 100).Return(7)
	calc := newCalculator(rnd)
	assert.Equal(t, 7, calc.Random())
}
```

另参考<https://github.com/jaytaylor/mockery-example>

# Goblin

Minimal and Beautiful Go testing framework <https://github.com/franela/goblin>

Goblin是一个小巧的测试框架，可以配合testing使用，内建了assert的一些方法
``` go
package test

import (
    "testing"
    . "github.com/franela/goblin"
)

func Test(t *testing.T) {
    g := Goblin(t)
    g.Describe("Numbers", func() {
        g.It("Should add two numbers ", func() {
            g.Assert(1+1).Equal(2)
        })
        g.It("Should match equal numbers", func() {
            g.Assert(2).Equal(4)
        })
        g.It("Should substract two numbers")
    })
}
```