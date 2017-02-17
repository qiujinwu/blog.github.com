---
title: Cookie跨域和单点登录
date: 2017-02-17 15:46:34
tags:
 - sso
 - cookie
categories:
 - 运营运维
---

若只涉及到子域名的cookie共享问题，cookie有一个domain的字段，例如aaa.ccc.com和bbb.ccc.com若要共享，设置domain为【.ccc.com】即可（注意.号不能省）

若是不同的域名，那就必须想其他办法

# 种Cookie
还是aaa.com和bbb.com。
1. aaa登录成功之后，服务器返回cookie
2. aaa继续调用一个枚举所有需要种cookie的url列表

接下来分成两种情况
1. 返回的地址可以直接登录，例如<http://bbb.com?token=fasdfasf&timestamp=123>。接下来使用jsonp的方式调用这个地址，完成bbb.com的域名种植，成功植入后token失效。为了安全起见，token需要设置有效期
2. 不用jsonp，返回的地址列表无敏感信息，继续在aaa.com发起一个<http://aaa.com/login_another?domain=bbb.com>的请求，在请求中重定向到<http://bbb.com?token=fasdfa>完成cookie种植


# 偷Cookie
bbb.com登录时，若本地没有cookie，就用jsonp的方式发起<http://aaa.com/stolen_cookie>的请求，通过aaa的服务器偷到aaa本机的所有cookie，然后用js设置到当前域名，最后再发起登录

也可以用stolen_cookie获取一个可以直接登录的token，然后使用这个token完成身份验证。

更为复杂的可以综合两种方式，比如iframe直接请求<http://aaa.com/stolen_cookie>，aaa服务器发现已经登录就重定向到带token的bbb.com，最后bbb就种好了cookie，在通过一些渠道通知父页面刷新。

# 参考
1. <http://www.cnblogs.com/showker/archive/2010/01/21/1653332.html>
2. <http://www.oschina.net/question/4873_18517>