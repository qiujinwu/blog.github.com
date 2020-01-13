---
title: Go编译时变量
date: 2020-01-13 00:00:00
tags:
 - golang
categories:
 - Golang
---

支持每次打包都动态更新版本号,编译时间之类的变量

```
-X importpath.name=value 
Set the value of the string variable in importpath named name to value. 
Note that before Go 1.5 this option took two separate arguments. 
Now it takes one argument split on the first = sign.
```

``` golang
package main

import (
	"fmt"
)

var buildstamp = ""

func main() {
	fmt.Printf("Build Time : %s\n", buildstamp)
}
```

``` bash
$export flags="-X main.buildstamp=`date -u '+%Y-%m-%d_%I:%M:%S%p'`"
$go build -ldflags "$flags" -o m m.go 
$./m
```

# 参考
1. <https://golang.org/cmd/link/>
