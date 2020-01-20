---
title: 关于配置文件的一些想法
date: 2020-01-20 00:00:00
categories:
 - 学习笔记
---

当前配置文件可选择的非常多, 从很早的ini、xml到后来的json、yaml, 以及最近讨论得比较多的toml、hcl等

总体而言, 现在越来越倾向于易与人阅读的格式, 例如yaml、toml、hcl用来结构化的符号已经非常少了, 对比此前的xml、json有明显的区别

具体怎么选择, 首先要从业务需求分析, 比如就几个key/value对, 甚至可以考虑用参数传入, 下面从几个方面考虑

Ini
``` ini
; 稍微复杂一点的单层嵌套结构

[c]
x = c.x
y = c.y

[d]
x = d.x
y = d.y
```
---
java properties
``` properties
#以下为服务器、数据库信息
dbPort = localhost
databaseName = mydb
dbUserName = root
dbPassword = root
#以下为数据库表信息
dbTable = mytable
#以下为服务器信息
ip = 192.168.0.9
```
---
json
``` json
{
    "a": "a",
    "b": "b",
    "c":{
        "x": "c.x",
        "y": "c.y"
    },
    "d":{
        "x": "d.x",
        "y": "d.y"
    },
    "e":[
        { "x":"e[0].x", "y":"e[0].y" },
        { "x":"e[1].x", "y":"e[1].y" }
    ]
}
```
---
yaml
``` yaml
b4: NuLL # string
b5: Null # null
c:
  x: c.x
  y: c.y
d:
  x: d.x
  y: d.y
```
---
toml
``` toml
a = "a"
b = "b"

c.x = "c.x"
c.y = "c.y"

[d]
x = "d.x"
y = "d.y"

[[e]]
x = "e[0].x"
y = "e[0].y"

[[e]]
x = "e[1].x"
y = "e[1].y"
```

## 机器友好&人友好
机器友好,意味着`表述能力强`, `读写效率高`, 如果有频繁的读写操作, 并且很少会需要人来修改, 那么此类较为合适

人友好,则表示`容易看懂`, 若是修改`不易出错`.

> 对于一个复杂 ( *特别是嵌套比较深* ) 的配置, 如果需要人来修改, 并非合适的选项, 做个配置页面会更好

像xml虽然强大, 但是解析/阅读都不易, 新项目用得越来越少了


## 注释友好
除非完全不和人打交道, 否则注释是配置中非常重要的一块内容

xml/json对注释支持就不太好, yaml/properties/toml支持`#注释`, 部分还支持行尾注释, 而hcl则支持C语言风格的注释

## 多行友好
在json中写多行需要手动加换行符, 既麻烦又难以阅读, yaml原生支持
``` yaml
key: |
  line1
  line2
```

toml进一步扩展了多行的支持

## 空格恐惧症
像yaml / toml / python 等通过空格缩进来确认层级的配置/语言, 容易因为多/少空格引发错误, 不过如今的编辑器都非常智能, 此类问题较为容易发现

减少结构化符号带来了空格维护问题, 过多的符号则影响内容的阅读, hcl则做了一个折中, 保留了json的大括号, 但是其他的符号适当的省略了

## 语法增强

所谓配置文件和编程语言相向而行, 越来越像对方了

### 数组
ini、java properties等对数组支持不是很好, 后续基本都是标配. 目前yaml/toml都支持整形/浮点/布尔/日期等常用类型

### 层级简化
``` toml
name = "Orange"
physical.color = "orange"
physical.shape = "round"
site."google.com" = true
```
``` json
{
  "name": "Orange",
  "physical": {
    "color": "orange",
    "shape": "round"
  },
  "site": {
    "google.com": true
  }
}
```

### 代码复用
对于一些共用的变量, 块状配置, 支持引用,  减少维护成本.  主流的配置格式暂未发现方案

有一些第三方的库可以支持, 例如<https://pypi.org/project/jsonref/>


### 错误检查
xml/json 有自己的校验框架,  例如[json schema](https://python-jsonschema.readthedocs.io/en/stable/), 但这并不是解释器的功能, 也并非必须, 而新的配置, 可能会做一些语法上的要求, 规避一些常见的坑, 比如toml字典里key不可重复

## 公司技术栈和语言文化
每个公司都有自己的技术栈, 比如阿里的Java, 那么具体用什么配置,很多时候会参考这个技术栈里面广泛使用的配置方案.  scalar就搞了一个[HOCON](https://github.com/lightbend/config/blob/master/HOCON.md)



+ <https://github.com/toml-lang/toml>
+ <https://github.com/hashicorp/hcl>
