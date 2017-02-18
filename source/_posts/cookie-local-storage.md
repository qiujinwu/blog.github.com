---
title: Cookie和Storeage对比
date: 2017-02-18 09:57:49
tags:
 - xss
 - cookie
 - crsf
categories:
 - 后端
---

如果只存储仅限前端使用的数据，毫无疑问，无必要使用cookie

Storage又区分LocalStorage（本地存储）和sessionStorage（会话存储），LocalStorage和sessionStorage功能上是一样的，但是存储持久时间不一样。
1. LocalStorage：浏览器关闭了数据仍然可以保存下来，并可用于所有同源（相同的域名、协议和端口）窗口（或标签页）永久存储，永不失效，除非手动删除
2. sessionStorage：数据存储在窗口对象中，窗口关闭后对应的窗口对象消失，存储的数据也会丢失。就是浏览器窗口关闭就失效了。

# 主要区别
1. cookie可以设置过期时间，domain，path等，storage不支持
2. cookie容量很小，storage相对较大，大概5M左右，而cookie只有几K
3. 默认cookie和storage都可以被js读写，这容易招致[XSS攻击](http://www.cnblogs.com/lovesong/p/5199623.html)，cookie支持[HttpOnly属性](http://www.cnblogs.com/alanzyy/archive/2011/10/14/2212484.html)，禁止js读写。（另一个有用的属性是**secure**）
4. cookie自动由浏览器带入服务器，或者由服务器下发，storage是纯客户端的东西，若要发送到服务器，必须用js提取并写入http query参数，form/json参数或者http头等正规渠道。
5. cookie自动提交的特性招致了[crsf攻击](http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html)

# XSS
XSS 全称“跨站脚本”，是注入攻击的一种。其特点是不对服务器端造成任何伤害，而是通过一些正常的站内交互途径，例如发布评论，提交含有 JavaScript 的内容文本。这时服务器端如果没有过滤或转义掉这些脚本，作为内容发布到了页面上，其他用户访问这个页面的时候就会运行这些脚本。

又区分
1. Stored XSS，持久化，代码是存储在服务器中的，所有访问的用户都受影响，波及方位广。
2. Reflected XSS，非持久化，需要欺骗用户自己去点击链接才能触发XSS代码。
3. DOM-based or local XSS，基于DOM或本地的XSS攻击。一般是提供一个**免费的wifi**，但是提供免费wifi的网关会往你访问的任何页面插入一段脚本或者是直接返回一个钓鱼页面，从而植入恶意脚本。这种直接存在于页面，无须经过服务器返回就是基于本地的XSS攻击。

### 防护
1. 对用户的输入进行处理，只允许输入合法的值，其它值一概过滤掉。
2. 对不可信的js来源进行完整性校验，现在前端经常使用cdn的js代码，若受到攻击就容易招致攻击

``` xml
<script src="https://example.com/example-framework.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```

# CRSF

CRSF是跨站请求伪造：Cross site request forgery，是一种对网站的恶意利用。它和XSS跨站脚本攻击的不同是，XSS利用站点内的信任用户；CRSF则是通过伪装来自信任用户的请求，来利用受信任的网站。举例就是：攻击者盗用了你的身份，以你的名义向第三方信任网站发送恶意请求，如发邮件，发短信，转账交易等。

### 防护
1. 勿使用get作更新操作
2. 服务器根据Referer字段等做一些简单的防护，参见[盗链](http://blog.qiujinwu.com/2017/02/17/stolen-chain/)
2. 在请求中放入攻击者所不能伪造的信息，并且该信息不存在于Cookie之中。通常在http头或者hidden表单中传递，可以从用js直接从cookie读取，或者直接从服务端返回，但这种情况下要配合referer防护使用。
3. 在请求中加入验证码，但增加了用户的使用成本，影响体验。