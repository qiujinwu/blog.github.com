---
title: 关于跨域
date: 2017-02-16 19:07:40
tags:
 - cors
 - jsonp
 - nginx
categories:
 - 后端
---

说到跨域，做过web开发的都知道怎么回事，本文只是做个笔记总结

常用的几种跨域的方式

# jsonp
jsonp解决跨域问题的原理是，浏览器的script标签是不受同源策略限制,你可以在你的网页中设置script的src属性问cdn服务器中静态文件的路径(图片也不受限)。那么就可以使用script标签从服务器获取数据，请求时添加一个参数为callbakc=?，?号时你要执行的回调方法。

服务器返回的数据类似于

``` javascript
callBack({
	"param1":true,
    "param2":["aa","bb"]
})
```

本质上就是传入的callback调用需要返回的json参数

### 不足
1. 只能用于get方法，无法传递大量数据，或者不适合传输部分数据
2. 如果出现错误，不会像http请求那样有状态码

ajax和jsonp这两种技术在调用方式上“看起来”很像，目的也一样，都是请求一个url，然后把服务器返回的数据进行处理，因此jquery和ext等框架都把jsonp作为ajax的一种形式进行了封装。

但是并不能用ajax去发送jsonp的请求，只能用dom操作动态构建dom元素来加载所谓的js。

jsonp虽然就比json多个p，但是并不一定要用json格式的数据作为回调的参数返回

# nginx反向代理作中继
既然浏览器禁止ajax的跨域行为，那么可以简单地然nginx在同域做个转发即可解决问题


# html5支持的cors
cors本质上就是服务器从头部返回适当的header，告诉浏览器是否允许访问，

### 浏览器请求
1. Origin 表示请求来自哪个源
1. Access-Control-Request-Method 该字段是必须的，用来列出浏览器的CORS请求会用到哪些HTTP方法。
1. Access-Control-Request-Headers 该字段是一个逗号分隔的字符串，指定浏览器CORS请求会额外发送的头信息字段。

### 服务器响应

1. Access-Control-Allow-Origin 该字段是必须的。它的值要么是请求时Origin字段的值，要么是一个*，表示接受任意域名的请求。
1. Access-Control-Allow-Credentials 该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为true，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可。
1. Access-Control-Expose-Headers 该字段可选。CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma。如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定。上面的例子指定，getResponseHeader('FooBar')可以返回FooBar字段的值。
1. Access-Control-Max-Age 该字段可选，用来指定本次预检请求的有效期，单位为秒。上面结果中，有效期是20天（1728000秒），即允许缓存该条回应1728000秒（即20天），在此期间，不用发出另一条预检请求。
1. Access-Control-Allow-Methods 该字段必需，它的值是逗号分隔的一个字符串，表明服务器支持的所有跨域请求的方法。注意，返回的是所有支持的方法，而不单是浏览器请求的那个方法。这是为了避免多次"预检"请求。
1. Access-Control-Allow-Headers 如果浏览器请求包括Access-Control-Request-Headers字段，则Access-Control-Allow-Headers字段是必需的。它也是一个逗号分隔的字符串，表明服务器支持的所有头信息字段，不限于浏览器在"预检"中请求的字段。

### 非简单请求 & 预检请求
简单请求的条件如下

```
（1) 请求方法是以下三种方法之一：
  HEAD
  GET
  POST
（2）HTTP的头信息不超出以下几种字段：
  Accept
  Accept-Language
  Content-Language
  Last-Event-ID
  Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain
```

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json。

非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。预检请求可能包含上述是那种字段，而其他普通请求只需要Origin即可，如果Origin指定的源，不在许可范围内，服务器会返回一个正常的HTTP回应。浏览器发现，这个回应的头信息没有包含Access-Control-Allow-Origin字段（详见下文），就知道出错了，从而抛出一个错误.





# 参考
1. <http://www.ruanyifeng.com/blog/2016/04/cors.html>
2. <http://www.cnblogs.com/dowinning/archive/2012/04/19/json-jsonp-jquery.html>
3. <http://www.jianshu.com/p/7257e7c60ef5>