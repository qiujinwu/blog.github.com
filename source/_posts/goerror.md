---
title: errors库
date: 2020-01-13 00:00:00
tags:
 - golang
categories:
 - Golang
---

# Pkg/errors

``` golang
package main

import (
	"fmt"

	"github.com/pkg/errors"
)

func Func() error {
	return fmt.Errorf("oring error")
}

func Func1() error {
	if err := Func(); err != nil {
		return errors.WithMessage(err, "Func1")
	}
	return nil
}

func Func2() error {
	err := Func1()
	if err != nil {
		return errors.Wrap(err, "Func2")
	}
	return nil
}

func main() {
	if err := Func2(); err != nil {
		fmt.Print(err)
		fmt.Println("\n------\n")
		fmt.Printf("%+v", err)
		return
	}
}
```

+ `New(message string)` 生成的错误，自带调用堆栈信息 // `errors.New`/`fmt.Errorf`
+ `WithMessage(err error, message string)`  只附加新的信息
+ `func WithStack(err error) error` //只附加调用堆栈信息
+ `func Wrap(err error, message string) error` //同时附加堆栈和信息

`func Cause(err error) error` 获取原始错误

## Print
> fmt.Printf

+ `%s,%v` 功能一样，输出错误信息，不包含堆栈
+ `%q` 输出的错误信息带引号，不包含堆栈
+ `%+v` 输出错误信息和堆栈

# juju/errors
``` golang
package main

import (
	"fmt"

	"github.com/juju/errors"
)

func Func() error {
	return fmt.Errorf("oring error")
}

func Func1() error {
	if err := Func(); err != nil {
		return errors.Annotate(err, "Func1")
	}
	return nil
}

func Func2() error {
	err := Func1()
	if err != nil {
		return errors.Trace(err)
	}
	return nil
}

func main() {
	if err := Func2(); err != nil {
		fmt.Print(err)
		fmt.Println("\n------\n")
		fmt.Printf("%+v", err)
		return
	}
}
```

# GOPROXY
``` bash
$go env -w GOPROXY=https://goproxy.cn,direct
$export GOPROXY=https://goproxy.cn
```



# 参考
1. <https://github.com/juju/errors>
2. <https://github.com/pkg/errors>

