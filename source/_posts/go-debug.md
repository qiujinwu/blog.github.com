---
title: Golang 调试
date: 2017-10-17 21:16:48
tags:
 - golang
categories:
 - Golang
---

# 编译选项

传递-gcflags "-N -l" 参数，这样可以忽略Go内部做的一些优化，聚合变量和函数等优化，这样对于GDB调试来说非常困难，所以在编译的时候加入这两个参数避免这些优化。

1. 对于单个go文件的，直接 【go build -gcflags "-N -l" main.go】
2. 对于基于package的，使用包路径  【go build -gcflags "-N -l" github.com/qjw/kelly/sample】,在当前目录直接.即可【go build -gcflags "-N -l" .】

# GDB

处理下断点时，注意函数的名称，其他都和C一致

以main函数为例，不能直接b main，这虽然会生效，但是实际上会陷入c库的main，而不是golang的main，务必使用包名+函数名
``` bash
king@king:/go/src/github.com/qjw/kelly/sample$ gdb sample 
(gdb) b main.main
Breakpoint 1 at 0x8bfe70: file /go/src/github.com/qjw/kelly/sample/main.go, line 42.
(gdb) l
28			Password: "",
29			DB:       3,
30		})
31		if err := redisClient.Ping().Err(); err != nil {
32			log.Fatal("failed to connect redis")
33		}

```

# [Delve](https://github.com/derekparker/delve)

安装指引 <https://github.com/derekparker/delve/blob/master/Documentation/installation/linux/install.md>，简单地就可以

``` bash
go get github.com/derekparker/delve/cmd/dlv
```

最终debug程序就是dlv，目前Gogland等很多IDE的调试器就使用dlv。

``` bash
king@king:/go/src/github.com/qjw/kelly/sample$ dlv exec ./sample 
Type 'help' for list of commands.
(dlv) b main.main
Breakpoint 1 set at 0x8bfe8b for main.main() ./main.go:42
(dlv) continue
> main.main() ./main.go:42 (hits goroutine(1):1 total:1) (PC: 0x8bfe8b)
    37:			log.Print(err)
    38:		}
    39:		return store
    40:	}
    41:	
=>  42:	func main() {
    43:		store := initStore()
```

自动从源码编译

``` bash
king@king:/go/src/github.com/qjw/kelly/sample$ dlv debug .
Type 'help' for list of commands.
(dlv) b main.main
Breakpoint 1 set at 0xa1d2db for main.main() ./main.go:42
(dlv) continue
> main.main() ./main.go:42 (hits goroutine(1):1 total:1) (PC: 0xa1d2db)
    37:			log.Print(err)
    38:		}
    39:		return store
    40:	}
    41:	
=>  42:	func main() {
```
