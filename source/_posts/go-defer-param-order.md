---
title: "Golang defer的坑,以及函数执行顺序差异"
date: 2018-07-16 16:14:19
tags:
 - golang
categories:
 - Golang
---

C/C++函数结果作为参数,会从右到左

``` cpp
#include <cstdio>

class B{
    public:
        int f(int i){
            printf("B %d\n",i);
            return i;
        }
};

class A{
    public:
        B f(int i){
            printf("A %d\n",i);
            B b;
            return b;
        }
};

void f(int a,int b){
    printf("C %d %d\n",a,b);
}


int main(){
    B b;
    A a;
    // 从右到左执行
    f(a.f(1).f(2),b.f(3));
    return 0;
}
```
``` bash
B 3
A 1
B 2
C 2 3
```

golang变成了从左到右

``` go
package main

import "fmt"

type B struct{}

func (*B) f(p int) int {
	fmt.Printf("B %d\n", p)
	return 0
}

type A struct{}

func (*A) f(p int) *B {
	fmt.Printf("A %d\n", p)
	return &B{}
}

func dd(*B, int) {
	fmt.Println("C")
}

func main() {
	a := &A{}
	b := &B{}
    // 在defer之前,会计算到保留最后一个函数,也就是a.f(1)会先执行
	defer a.f(1).f(2)
    // 从左到右执行
	defer dd(a.f(3), b.f(4))
    // 匿名函数,defer统一执行
	defer func() {
		a.f(5).f(6)
	}()
	fmt.Println("main")
}
```
```
A 1
A 3
B 4
main
A 5
B 6
C
B 2
```