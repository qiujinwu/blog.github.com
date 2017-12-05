---
title: telegraf/influxdb/grafana初探
date: 2017-12-05 19:07:03
tags:
 - influxdb
categories:
 - 学习笔记
---

# 安装
influxdb/grafana直接docker运行最简单

``` bash
// 下载镜像
docker pull influxdb
docker pull grafana/grafana
# 运行（第一次）
docker run -p 8086:8086 --name influxdb -v $PWD:/var/lib/influxdb influxdb
docker run -p 3000:3000 --name grafana grafana/grafana
# 创建了容器之后，由于指定了自定义name，后续可以根据这个name直接启动
docker start influxdb
docker start grafana
```
telegraf直接安装

下载地址 <https://github.com/influxdata/telegraf/releases>
``` bash
sudo dpkg -i /home/king/telegraf_1.4.5-1_amd64.deb
# 安装客户端
sudo apt-get install influxdb-client
# docker也是可以的
# https://hub.docker.com/_/telegraf/
```

# influxdb
## [基本概念](https://docs.influxdata.com/influxdb/v0.9/concepts/key_concepts)

1. dababase 数据库
2. [measurement](https://docs.influxdata.com/influxdb/v0.9/concepts/key_concepts/#measurement)，数据库表
3. points,类比数据表行
4. time, 时间戳，由于是时序数据库，每个数据表都会有一列时间戳的列
5. field, 字段数据，没有索引，通常根据时间来呈现
6. tag, 字段的标签，有索引，可以以此进行过滤
7. [series](https://docs.influxdata.com/influxdb/v0.9/concepts/key_concepts/#series), a series is the collection of data that share a retention policy, measurement, and tag set，

## field和tag
两者除了索引的区别外，本质上是定位的差异，举个例子，有数据

``` bash
时间戳 姓名 城市 收入 支出
```

`收入`、`支出`定位于field，没有实际的规律，一般不会用它来做顾虑（传统的关系数据库，常见的可以根据范围过滤搜索），而`姓名`、`城市`定位于Tag，可以理解为这一行（`point`）的属性，我们会有诸如

1. 所有人的收入曲线图
2. 广州、深圳等各个城市的收入，支出曲线图
3. 甚至只看广州的张三的支出曲线图

`Tag`一般在一个有限的集合中选取，而不是`field`那样无明显的规律


## 写入数据
``` bash
# 创建数据库
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE telegraf"
# 写入数据，最后的时间戳可以省略，自动用当前时间(test: measurement名称）
curl -i -XPOST 'http://localhost:8086/write?db=telegraf' --data-binary  \
	'test,tag1=server01,tag2=us-west field1=0.64,field2=11 1434055562000000000'
```

也可以使用cli工具
``` bash
king@king:~/tmp$ influx 
Visit https://enterprise.influxdata.com to register for updates,
   InfluxDB server management, and monitoring.
Connected to http://localhost:8086 version 1.4.2
InfluxDB shell 0.10.0
> show databases;
name: databases
---------------
name
_internal
telegraf

> use telegraf;
Using database telegraf
> show measurements;
name: measurements
------------------
name
money
processes
swap
system
test

> show series from money
key
money,city=上海,name=张三
money,city=北京,name=刘七

> select * from money limit 2
name: money
-----------
time			city	in	name	out
1512445152435671225	深圳	28	赵六	1
1512445153519585060	上海	76	李四	95

```

## 简单的数据源

不依赖于telegraf采集器，在程序中，也有非常多的api可选,参见<https://docs.influxdata.com/influxdb/v0.9/clients/api/>

``` bash
#!/bin/bash

url="http://localhost:8086/write?db=telegraf"
names=('张三' '李四' '王五' '赵六' '刘七')
citys=('深圳' '广州' '北京' '上海' '成都')

doWrite(){
    curl -i -XPOST "${url}" --data-binary "money,name="${1}",city="${2}" in="${3}",out="${4}""
}

for((i=0;i<10000;i++))
do
    in=$(($(echo $RANDOM) % 100))
    out=$(($(echo $RANDOM) % 100))

    cityid=$(($(echo $RANDOM) % 5))
    nameid=$(($(echo $RANDOM) % 5))
    doWrite ${names[nameid]} ${citys[cityid]} "${in}" "${out}"
    sleep 1
done
```

# [Telegraf](https://github.com/influxdata/telegraf)

telegraf是一个数据采集器，自身有一些基本的采集功能，比如cpu、内存等。支持插件的设计使得采集范围非常广泛，支持列表见<https://github.com/influxdata/telegraf#input-plugins>

具体的配置见官方文档，为了对接influxdb，我们先需要修改`/etc/telegraf/telegraf.conf`
``` ini
# Configuration for influxdb server to send metrics to
[[outputs.influxdb]]
  ## The full HTTP or UDP URL for your InfluxDB instance.
  ##
  ## Multiple urls can be specified as part of the same cluster,
  ## this means that only ONE of the urls will be written to each interval.
  # urls = ["udp://localhost:8089"] # UDP endpoint example                      
  urls = ["http://localhost:8086"] # required
  ## The target database for metrics (telegraf will create it if not exists).   
  database = "telegraf" # required  
```

直接运行`telegraf`命令启动服务器（为什么没有/etc/init.d/xx）

# [grafana](https://grafana.com/)

grafana本身支持很多后端，包括influxdb，它是启动之后再进行参数配置。

首先配置数据源,influxdb参考<http://docs.grafana.org/features/datasources/influxdb/>

然后添加dashboard-panel。在panel的metrics里有各种参数（本质上就是一个sql的语义），以及一些聚合函数支持

完善权限控制

这三个东西和elk技(全)术(家)栈(桶)有些有意思的对比，[这里](http://www.infoq.com/cn/articles/grafana-vs-kibana-the-key-differences-to-know)有一个grafana和kibana的比较

# 参考
1. <http://www.jianshu.com/p/dfd329d30891>
