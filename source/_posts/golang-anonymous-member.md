---
title: Golang 匿名对象指针和对象的区别
date: 2017-7-3 12:23:11
tags:
 - golang
categories:
 - Golang
---


都说**一图胜千言**，有代码就不废话了

``` go
package main

import (
	"fmt"
)

type Animal struct {
	Name string
}
type Persion struct {
	Animal
}
type Ppersion struct {
	*Animal
}

func main() {
	animal := Animal{Name: "Cat"}
	persion := Persion{animal}
	ppersion := Ppersion{&animal}
	fmt.Println("Animal:" + animal.Name)
	fmt.Println("Persion:" + persion.Name)
	fmt.Println("PPersion:" + ppersion.Name)

	animal.Name = "Dog"
	fmt.Println("------------我是卖萌分割线------------")
	fmt.Println("Animal:" + animal.Name)
	fmt.Println("Persion:" + persion.Name)
	fmt.Println(persion.Animal == animal)
	fmt.Println("PPersion:" + ppersion.Name)
	fmt.Println(ppersion.Animal == &animal)
}
```
``` bash
Animal:Cat
Persion:Cat
PPersion:Cat
------------我是卖萌分割线------------
Animal:Dog
Persion:Cat
false
PPersion:Dog
true
```

# struct/interface转换
下面的代码会报错，因为Stduent未实现People的接口，实现People接口的是*People，所以改法有两种

1. var peo People = &Stduent{} # 用指针赋值给People
2. func (stu Stduent) Speak(think string) (talk string) # Stduent对象实现Speak方法


``` go
package main

import (
	"fmt"
)

type People interface {
	Speak(string) string
}

type Stduent struct{}

func (stu *Stduent) Speak(think string) (talk string) {
	if think == "bitch" {
		talk = "You are a good boy"
	} else {
		talk = "hi"
	}
	return
}

func main() {
	var peo People = Stduent{}
	think := "bitch"
	fmt.Println(peo.Speak(think))
}
```

# 继承和组合
下面的代码调用的是People.ShowB，因为go没有类似于C++的多态机制

>Go中没有继承！ 没有继承！没有继承！是叫组合！组合！组合！
这里People是匿名组合People。被组合的类型People所包含的方法虽然升级成了外部类型Teacher这个组合类型的方法，但他们的方法(ShowA())调用时接受者并没有发生变化。
这里仍然是People。毕竟这个People类型并不知道自己会被什么类型组合，当然也就无法调用方法时去使用未知的组合者Teacher类型的功能。
因此这里执行t.ShowA()时，在执行ShowB()时该函数的接受者是People，而非Teacher。具体参见[官方文档](https://golang.org/doc/effective_go.html#Embedding)

``` go
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}
```

# 参考
1. <https://segmentfault.com/q/1010000002687684>
2. <https://yushuangqi.com/blog/2017/golang-mian-shi-ti-da-an-yujie-xi.html>