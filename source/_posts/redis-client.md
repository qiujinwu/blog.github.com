---
title: Redis客户端
date: 2017-02-15 15:21:31
tags: redis
categories:
 - 开发环境
---

redis有官方的客户端[redis-cli](https://redis.io/topics/rediscli)，使用也很方便，不过有些时候GUI界面的客户端也是不错的选择。关于redis-cli用法，可参考

1. <https://gnuhpc.gitbooks.io/redis-all-about/Operation/redis-cli.html>
2. <https://www.kancloud.cn/thinkphp/redis-quickstart/36132>
3. <https://maoxian.de/2015/08/1342.html>

在Windows下有一个用java写的客户端[RedisClient](https://github.com/caoxinyu/RedisClient)，直接用下面的命令即可运行】

``` bash
java -jar redisclient-win32.x86.2.0.jar
```

虽说用java，但是用到了一些Windows特有的库，所以并不支持其他平台使用。

在Linux下，有一个[redisdesktop](http://docs.redisdesktop.com/en/latest/install/)的客户端程序，用QT编写，支持多个平台。本身开源，所以直接[build from source](http://docs.redisdesktop.com/en/latest/install/#build-on-linux)。（*需要翻墙*）

``` bash
# Get source code:
git clone --recursive https://github.com/uglide/RedisDesktopManager.git -b 0.8.0 rdm && cd ./rdm
# ubuntu
cd src/
./configure
source /opt/qt56/bin/qt56-env.sh && qmake && make
# 最终生成的文件路径 bin/linux/release/rdm

# 如果需要安装
sudo make install
```

也有不少web应用，推荐[redis-commander](https://github.com/joeferner/redis-commander)，官方的安装方法很简单

``` bash
# 安装
$ npm install -g redis-commander
# 运行
$ redis-commander

#
#
king@king:/$ redis-commander --redis-db=1
Using default configuration.
No Save: true
listening on  0.0.0.0 : 8081
Redis Connection localhost:6379 Using Redis DB #0
```

这种程序较原生客户端体验稍差，默认情况下，用浏览器打开<http://127.0.0.1:8081>即可访问，这里有个bug，不管指定哪个redisdb，打印的日志和web ui都是db0，但实际数据是对的。

