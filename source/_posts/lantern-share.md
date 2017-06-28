---
title: 蓝灯局域网共享
date: 2017-6-22 19:52:33
tags:
 - proxy
categories:
 - 运营运维
 - 代理
---

# [蓝灯](https://getlantern.org/)
**蓝灯** github <https://github.com/getlantern/lantern>，其他页面
1. <https://getlantern.org/>
2. <http://s3.amazonaws.com/urtuz53txrmk9/index.html>

下载For windows版本，安装，我的版本是3.7.2，默认绑定的127.0.0.1主机，使用了三个端口，一个用于ui，一个用于socks5代理，一个用于http代理。

windows下会自动设置系统代理为lantern的http代理，也可以用**SwitchyOmega**之类的工具只代理浏览器

具体使用的端口可以从配置文件【C:\Users\xxx\AppData\Roaming\Lantern\setting.yaml】文件中找到，key分别是addr、uiAddr、socksAddr。（*不同的版本，配置文件可能有出入，端口也有出入*）

另一种方法是在打开lantern的时候自动打开的浏览器页面查看，具体是点击左上角的按钮 - 设置 -  高级设置（addr、socksAddr），uiAddr从浏览器地址栏就可以知道。

默认情况下，可以上网，免费账户每个月500M高速流量

# 共享
为了共享给局域网使用，特别是版本欠缺的iphone，由于默认绑定的127.0.0.1，局域网其他主机是无法访问socks5和http代理的，最简单直接的办法就是让其绑定0.0.0.0，google了网上教程，发现都无效，可能新版本都修复了那些漏洞，毕竟任意让你共享，收费版就玩不下去了。

直接日它不行那就取巧，将local的代理forward（中继）到一个绑定了0.0.0.0的其他端口

具体使用的工具是**[privoxy](http://www.privoxy.org/)**，一个成熟稳定的代理工具，支持Windows，我下载的版本是**[3.0.10](http://www.privoxy.org/sf-download-mirror/Win32/)**

安装之后，找到【c:\Program files(x86)\Privoxy】的config.txt，将**listen-address**绑定的IP从127.0.0.1改成0.0.0.0，另外增加forward配置，将本机的lantern http代理forward到本机的8118端口
``` ini
listen-address  0.0.0.0:8118
forward / 127.0.0.1:49763
```

我尝试过用**【forward-socks5(t) / 127.0.0.1:49764 .】**来重定向socks5代理，发现不行，只好重定向http代理了。
