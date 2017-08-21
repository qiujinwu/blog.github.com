---
title: Golang 可选类型
date: 2017-8-20 22:06:13
tags:
 - golang
categories:
 - Golang
---

对于struct，有些字段可能是空值，有些字段不存在（这是两种含义），若对于不允许空值的场景，空值可以表示不存在的含义。

1. golang声明之后会赋一个默认值（区别C/C++）
2. 用万能的指针可以区分空值和不存在，不过存在**Null Pointer Exception**的问题

以下内容参考**<http://www.markphelps.me/2017/08/20/optional-types-with-go-generate.html>**

既然不用指针，就必须有一个标志表示**是否存在**，对象定义如下

``` go
package optional

type String struct {
	string  string
	present bool
}

func EmptyString() String {
	return String{}
}

func OfString(s string) String {
	return String{string: s, present: true}
}

func (o *String) Set(s string) {
	o.string = s
	o.present = true
}

func (o *String) Get() string {
	return o.string
}

func (o *String) Present() bool {
	return o.present
}
```

由于golang没有语言层面的模板（类比C++的[template](http://www.cplusplus.com/doc/oldtutorial/templates/)），而type却有很多种，所以需要有一种机制能快速给一个对象生成上述的代码。

利用[go template](https://golang.org/pkg/html/template/)机制和[go generate](https://blog.golang.org/generate)比较容易实现

模板定义

``` django
package {{ .PackageName }}

// {{ .OutputName }} is an optional {{ .TypeName }}
type {{ .OutputName }} struct {
    {{ .TypeName | unexport }} {{ .TypeName }}
    present bool
}

// Empty{{ .OutputName | title }} returns an empty {{ .PackageName }}.{{ .OutputName }}
func Empty{{ .OutputName | title }}() {{ .OutputName }} {
    return {{ .OutputName }}{}
}

// Of{{ .TypeName | title }} creates a {{ .PackageName }}.{{ .OutputName }} from a {{ .TypeName }}
func Of{{ .TypeName | title }}({{ .TypeName | first }} {{ .TypeName }}) {{ .OutputName }} {
    return {{ .OutputName }}{ {{ .TypeName | unexport }}: {{ .TypeName | first }}, present: true}
}

// Set sets the {{ .TypeName }} value
func (o *{{ .OutputName }}) Set({{ .TypeName | first }} {{ .TypeName }}) {
    o.{{ .TypeName | unexport }} = {{ .TypeName | first }}
    o.present = true
}

// Get returns the {{ .TypeName }} value
func (o *{{ .OutputName }}) Get() {{ .TypeName }} {
    return o.{{ .TypeName | unexport }}
}

// Present returns whether or not the value is present
func (o *{{ .OutputName }}) Present() bool {
    return o.present
}
```

代码定义

``` go
import (
	"time"
	"strings"
	"html/template"
	"bytes"
	"io/ioutil"
	"log"
)

type data struct {
	Timestamp   time.Time
	PackageName string
	TypeName    string
	OutputName  string
}

var (
	funcMap = template.FuncMap{
		"title": strings.Title,
		"first": func(s string) string {
			return strings.ToLower(string(s[0]))
		},
		"unexport": func(s string) string {
			return strings.ToLower(string(s[0])) + string(s[1:])
		},
	}
)

func main() {
	temp,err := ioutil.ReadFile("src/github.com/qjw/test/a.tpl")
	if err != nil{
		log.Panic(err)
		return
	}


	t := template.Must(template.New("").Funcs(funcMap).Parse(string(temp)))


	var buf bytes.Buffer
	err = t.Execute(&buf, data{
		Timestamp:time.Now(),
		PackageName: "main",
		TypeName: "int",
		OutputName: "Int",

	})
	if err != nil {
		log.Panic(err)
		return
	}

	err = ioutil.WriteFile("src/github.com/qjw/test/a_gen.go", buf.Bytes(), 0644)
	if err != nil {
		log.Fatalf("writing output: %s", err)
	}
}
```

也可以将其作为一个command，然后用[go generate](https://blog.golang.org/generate)批量生成类型代码，作者的代码在<https://github.com/markphelps/optional/blob/master/cmd/optional/main.go>