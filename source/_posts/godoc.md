---
title: Golang文档
date: 2017-6-22 20:23:16
tags:
 - golang
categories:
 - Golang
---

# Godoc
Godoc 的概念同 Python 的 [Docstring](http://www.python.org/dev/peps/pep-0257/) 和 Java 的 [Javadoc](http://www.oracle.com/technetwork/java/javase/documentation/index-jsp-135444.html)类似，但是设计上更为简单。

如果需要将注释转化成HTML形式的文档，Godoc用户还需要掌握一些额外的格式化规则：

1. 段落以空行格开
1. 预格式化的文档应该缩进（参考gob包的[doc.go](http://golang.org/src/pkg/encoding/gob/doc.go)）
1. URL将被转化为HTML链接，无需其它的特殊标记

## go doc
go doc命令可以打印附于Go语言程序实体上的文档。我们可以通过把程序实体的标识符作为该命令的参数来达到查看其文档的目的。

``` bash
king@king:~/code/go/src/test$ godoc fmt Printf
use 'godoc cmd/fmt' for documentation on the fmt command 

func Printf(format string, a ...interface{}) (n int, err error)
    Printf formats according to a format specifier and writes to standard
    output. It returns the number of bytes written and any write error
    encountered.
```
``` bash
king@king:~/code/go/src/test$ go doc fmt Printf
func Printf(format string, a ...interface{}) (n int, err error)
    Printf formats according to a format specifier and writes to standard
    output. It returns the number of bytes written and any write error
    encountered.
```
也可以处理当前目录下的模块
``` bash
king@king:~/code/go/src/test$ go doc Add2
func Add2(a, b int) int
    一个加法实现 返回a+b的值
```
``` bash
king@king:~/code/go/src/test$ go doc bb
package bb // import "test/bb"

提供的常用库，有一些常用的方法，方便使用

提供的常用库，有一些常用的方法，方便使用1222

func Add(a, b int) int
func Add2(a, b int) int
```

参数说明

| 标记名称 | 标记描述 |
|--------|--------|
|   -c     |    加入此标记后会使go doc命令区分参数中字母的大小写。默认情况下，命令是大小写不敏感的。    |
|   -cmd     |   加入此标记后会使go doc命令同时打印出main包中的可导出的程序实体（其名称的首字母大写）的文档。默认情况下，这部分文档是不会被打印出来的。      | 
|   -u     |    加入此标记后会使go doc命令同时打印出不可导出的程序实体（其名称的首字母小写）的文档。默认情况下，这部分文档是不会被打印出来的。       |

## godoc
```bash
king@king:~/code/go/src/test$ godoc fmt
use 'godoc cmd/fmt' for documentation on the fmt command 

PACKAGE DOCUMENTATION

package fmt
    import "fmt"

    Package fmt implements formatted I/O with functions analogous to C's
    printf and scanf. The format 'verbs' are derived from C's but are
    simpler.
```
``` bash
king@king:~/code/go/src/test$ godoc fmt Printf
use 'godoc cmd/fmt' for documentation on the fmt command 

func Printf(format string, a ...interface{}) (n int, err error)
    Printf formats according to a format specifier and writes to standard
    output. It returns the number of bytes written and any write error
    encountered.
```
``` bash
king@king:~/code/go/src/test$ godoc fmt Printf Println
use 'godoc cmd/fmt' for documentation on the fmt command 

func Printf(format string, a ...interface{}) (n int, err error)
    Printf formats according to a format specifier and writes to standard
    output. It returns the number of bytes written and any write error
    encountered.

func Println(a ...interface{}) (n int, err error)
    Println formats using the default formats for its operands and writes to
    standard output. Spaces are always added between operands and a newline
    is appended. It returns the number of bytes written and any write error
    encountered.
```

最牛叉的是http命令，可以直接打开浏览器看文档和代码，生成的文档包含本机GOPATH下的所有代码
``` bash
godoc -http=:6060
```

# swagger
对于Resful Api，用swagger ui可以很好的组织和测试接口，请参考项目<https://github.com/qjw/go-swagger-doc>

# Doxygen
Doxygen是一种开源跨平台的，以类似JavaDoc风格描述的文档系统，完全支持C、C++、Java、Objective-C和IDL语言，部分支持PHP、C#。注释的语法与Qt-Doc、KDoc和JavaDoc兼容。

参考旧文<http://qjw.qiujinwu.com/blog/2013/06/02/doxygen_study>

# 参考
1. <http://octman.com/blog/2014-02-24-godoc-documenting-go-code/>
2. <http://wiki.jikexueyuan.com/project/go-command-tutorial/0.5.html>
3. <https://mikespook.com/2011/04/%E3%80%90%E7%BF%BB%E8%AF%91%E3%80%91godoc%EF%BC%9A%E6%96%87%E6%A1%A3%E5%8C%96-go-%E4%BB%A3%E7%A0%81/>