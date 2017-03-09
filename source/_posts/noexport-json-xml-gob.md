---
title: Struct未导出变量序列化时的坑
date: 2017-03-09 14:26:27
tags:
 - golang
categories:
 - 后端
---

Go默认使用大小写来区分变量/函数是否导出，所以如果一个Struct需要对外使用，但是又未导出就会出现意想不到的bug

特别是常用的编解码（序列化/反序列化），json，xml，gob等

``` go
package main
import (
	"fmt"
	"encoding/json"
)
type MyData struct {
	One int
	two string
}
func main() {
	in := MyData{1,"two"}
	fmt.Printf("%#v\n",in) //prints main.MyData{One:1, two:"two"}
	encoded,_ := json.Marshal(in)
	fmt.Println(string(encoded)) //prints {"One":1}
	var out MyData
	json.Unmarshal(encoded,&out)
	fmt.Printf("%#v\n",out) //prints main.MyData{One:1, two:""}
}
```

为了避免大写在某些格式不符合规范的情况，通常这些框架都支持自定义序列化的tag，例如

``` go
type MyData struct {
	One int `json:"one"`
	Two string `json:"two"`
}
```

参考 <http://colobu.com/2015/09/07/gotchas-and-common-mistakes-in-go-golang/#>