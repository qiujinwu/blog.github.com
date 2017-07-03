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

# 参考
1. <https://segmentfault.com/q/1010000002687684>