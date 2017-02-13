---
title: 翻墙的一些工具
date: 2017-02-13 16:52:33
tags:
 - proxy
categories:
 - 运营运维
 - 代理
---

目前代理shadowsocks客户端连接，后者提供sock5代理，在Windows下可以直接全局代理，Linux比较麻烦。各种平台客户端见<https://shadowsocks.org/en/download/clients.html>

对于chrome浏览器，可以安装插件[switchyomega](https://github.com/FelisCatus/SwitchyOmega/releases)，以前有个[SwitchySharp](https://switchysharp.com/)
> SwitchyOmega, 下一代 SwitchySharp 升级版已经完成了。新版完全免费，只希望大家能够继续支持。
P.S. 别担心，更新后可自动升级设置，无须再次手动配置。

firefox可以使用[foxyproxy](https://addons.mozilla.org/zh-cn/firefox/addon/foxyproxy-standard/)

## 终端使用代理
### proxychains
注意配置中不要画蛇添足，在**ProxyList**一节中添加sock4

``` bash
$ apt-get install proxychains

$ cat /etc/proxychains.conf | sed "/^#/d" | sed "/^$/d"
strict_chain
chain_len = 1
proxy_dns 
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
socks5  127.0.0.1   1080
```

使用方法
``` bash
proxychains curl https://www.twitter.com/
proxychains git push origin master
```

## sock5代理转http代理
``` bash
$ sudo apt-get install polipo

$ cat /etc/polipo/config | sed "/^#/d" | sed "/^$/d"
logSyslog = true
logFile = /var/log/polipo/polipo.log
proxyAddress = "0.0.0.0"
socksParentProxy = "127.0.0.1:1080"
socksProxyType = socks5
chunkHighMark = 50331648
objectHighMark = 16384
serverMaxSlots = 64
serverSlots = 16
serverSlots1 = 32

$ sudo /etc/init.d/polipo restart
$ export http_proxy="http://127.0.0.1:8123/"
```

## 其他
1. <https://www.domecross.com/>
2. <https://crola.xyz/>
3. <https://shadowsocks.org/en/download/clients.html>

## 参考
1. <https://blog.xinshangshangxin.com/2015/06/21/linux%E7%BB%88%E7%AB%AF%E7%BF%BB%E5%A2%99/>
2. <https://jingsam.github.io/2016/05/08/setup-shadowsocks-http-proxy-on-ubuntu-server.html>