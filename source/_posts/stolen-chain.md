---
title: 关于盗链
date: 2017-02-17 08:33:18
tags:
 - 盗链
categories:
 - 运维
---

防盗链本质上就是要验证来源，防止对未许可、未授权的请求提供服务，根据验证身份的数据来源可以简单地分为两类
1. 直接从请求方主动携带的字段中提取，最常用的就是Http 头**Referer**字段，一些严格的验证包括验证**IP地址**，**User Agent**等。
2. 验证信息需要事先从服务端获取或者按照服务端约定的规则提供

# Referer
这种方式最流行，最简单，对服务器影响小，一般的cdn（腾讯云，阿里云cdn）都提供Referer防盗链机制，具体可以分为黑名单和白名单，通常支持通配符或者正则表达式，此外一个特例就是是否允许空的Referer。

一般不推荐不允许Referer为空，因为防盗链做得过于严密对于真实用户也是不方便的，比如隐私模式或HTTPS浏览时，就不存在Referer。对于一些特殊资源比如音视频，使用第三方播放器也可能不存在Referer。

常用的后端框架例如JavaSpring,NodeExpress,PythonFlask都可以找到基于Referer的防盗链中间件（过滤器），nginx也有插件<http://www.ttlsa.com/nginx/nginx-referer/>

## 破解
要破解首先要明确服务提供方的规则，如果对方允许空Referer，那么将图片dom元素置于frame中即可。一种更好的办法是用服务器中转，在服务器中清掉Referer甚至伪造，但带来额外的服务器开销。

nginx同样可以实现这样的功能

# 请求方提供授权信息
一般理解，授权就是需要登录的那种，在需要授权的web系统中，可以一并对图片等静态资源作授权（*而不仅仅是ajax接口或者web网页路径，虽然静态资源通常不会加入授权访问*）。

## Cookie
若只允许资源在本域访问，可以在非资源请求时下发携带授权信息的cookie，而服务器在**资源请求中验证cookie**，这种方式也很简单，但是若需要授权第三方就比较麻烦。

要实现第三方授权，可以参考**跨域cookie共享**

这种方案仍然可以用非浏览器的方式获取，

## Url Token加超时
若需要授权第三方使用，可以给他提供一个密钥（对称）或者公钥（非对称），在请求我们资源时，url必须携带token和timestamp,token通过url和timestamp加密得来（具体算法呵呵），

### 根据约定的规则
在我方验证时，需确保token正确，以及时间戳和当前时间戳误差在一定时间内（比方5分钟）。之所以要验证误差，是为了避免因无超时第三方偷懒使用一个固定的地址，而这种固定地址泄漏之后就一直可访问，关于url加密，可以用nginx插件<http://www.ttlsa.com/nginx/nginx-modules-secure_link/>

## 自己提供Token
这常见于下载站，请求方需要事先获取一个下载url，这个url同样包含一个token和时间戳，时间戳表示过期时间，这样可以避免url一直可用。

或者简单的，生成一个验证码，并且把验证码放入cookie中，下载时，必须携带这个验证码作为参数，服务器只需要简单的验证cookie中的验证码和参数是否一致，然后删除cookie(使用内存cookie或者超时很短也可以)，确保立即失效。

# 会话
自行/协商生成url 加 超时是个不错的防盗链方法，但有时超时并不一定满足需求，比如专门提供内嵌视频的视频服务提供商，此时商家可能只需要简单地内嵌一个frame网页。

这种情况下，视频的地址仍然是服务提供商动态生成，比如<http://video.example.com/a.avi?key=fasdfasd>，由于没有超时，这个地址一直可用，为了避免盗链，服务器后端就需要保存会话。
1. 在第一次访问时，用参数的key作为key，通过Ip，User Agent，Referer等计算value并将键值对存入数据库或者redis中，
2. 若key存在，就验证value是否匹配

这里的key可以看做跟软件序列号一样，可以通过一个算法批量产生，并且能够自验证有效性。以降低非法key对后端数据库造成的查询压力。

# 参考
1. <https://blog.goquxiao.com/posts/2015/06/13/fang-dao-lian-zi-liao-zheng-li/>
2. <http://oneoo.com/2008/09/03/uri-the-use-of-secret-technology-for-the-key-anti-daolian/>
3. <https://onebitbug.me/2013/10/10/use-nginx-lua-module-to-prevent-hotlinking/>


