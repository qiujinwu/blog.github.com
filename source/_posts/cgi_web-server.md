---
title: CGI网关服务器漫谈
date: 2017-7-22 21:50:49
tags: wcgi
categories:
 - 后端
---

作为一名C/C++转后端的程序员，习惯于socket的裸（*在tcp/ip下面操作系统还隐藏了很多细节，这里姑且不论*），会纠结一个问题，为什么python还需要wsgi服务器夹在中间呢？

在一个web请求到达实际的业务逻辑，需要（但不限于）经过
1. 处理tcp连接，读写二进制数据
2. 解析http协议，并判断合法性
3. 检查参数合法性
4. 检查权限/访问控制
5. 安全防护检查
6. 特定语言的参数序列化（绑定）

而业务逻辑处理之后，又需要（但不限于）经过
1. 反序列化数据（比如对象转json/xml）
2. 格式化输出（比如html模板引擎）

除了各种中间服务器，还有各种web框架，例如Python [flask](http://flask.pocoo.org/)等

对于socket处理，Http(s)协议解析，[nginx](https://nginx.org/en/)/[apache](http://httpd.apache.org/)等成熟服务器是强项，由于是通用的服务器，并不会和特定的语言绑定，所以要和用特定程序语言实现的web业务逻辑配合工作，就需要一定的协议来交互，这就产生了CGI。

并不是所有的后端语言都需要独立服务器来配合处理网络和http协议，Golang就是一个例外，Golang内置的[Http库](https://golang.org/pkg/net/http/)非常完善（也有很高效的第三方库，比如[fasthttp](https://github.com/valyala/fasthttp)），所以Go web框架大都把这块直接包进去了。
> 这种做法在部署上有更好的灵活性和便捷性，特别是配合[docker](https://www.docker.com/)，对比[spring boot](https://projects.spring.io/spring-boot/)打包[jetty](http://www.eclipse.org/jetty/)直接运行

对于其他语言，例如python,通常会采用Http <--> WSGI <--> WEB框架 <--> 业务代码 的方式
> 并非python无法实现类似与Golang的大一统集成方案，理论上什么语言都是可行的，不过基于性能上的考虑，或者避免重复造轮子等。

这种方案利用了各自的优势，协同完成整个事情，但他们并非物理分割的实体，比如[uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/)或者[gunicorn](http://gunicorn.org/)就实现了http服务器和wsgi协议处理，

在很多文章中，例如【[Flask+uwsgi+Nginx](http://www.jianshu.com/p/84978157c785)】，会再附加一个nginx在最前面，这更多的是为了扩展其他一些功能，比如
1. 反向代理
2. 负责均衡
3. 代理静态资源
4. https
5. ...

> nginx负载均衡在应用层处理，在一些高性能场景，可能需要特殊定制的负载均衡服务器，直接在内核的ip层/数据链路层作处理

scgi的一个简单实现如[wsgiref.simple_server](https://docs.python.org/2/library/wsgiref.html#module-wsgiref.simple_server)，基于它很容易写出一个简单的[Python web程序](http://blog.ez2learn.com/2010/01/27/introduction-to-wsgi/)，或者扩展成一个web框架

``` python
from wsgiref.simple_server import make_server

def my_app(environ, start_response):
    """a simple wsgi application"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return [u"你好! 歡迎來到Victor的第一個WSGI程式".encode('utf8')]

httpd = make_server('', 8000, my_app)
print("Serving on port 8000...")
httpd.serve_forever()
```

借助[uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/)也很[容易实现](http://www.bjhee.com/nginx-uwsgi.html)
``` python
def application(environ, start_response):
    status = '200 OK'
    output = 'Hello World!'

    response_headers = [('Content-type', 'text/plain'),
                        ('Content-Length', str(len(output)))]
    start_response(status, response_headers)

    return [output]
```

``` bash
pip install uwsgi
king@king:~/code/bug$ uwsgi --version
2.0.15
king@king:~/code/bug$ uwsgi --http :9090 --wsgi-file server.py
```

wcgi利用了语言的特性，直接通过调用和回调的方式实现高效请求转发和回传，wcgi和业务代码就在同个进程中，而最早的cgi并非这样

最早的[CGI](https://shenwang.blog.ustc.edu.cn/nginx%E5%92%8Cuwsgi%E9%80%9A%E4%BF%A1%E6%9C%BA%E5%88%B6/)是很灵活的，收到请求后CGI网关会启动子进程，并通过IPC（通常的做法是将stdin/stdout对接到输入输出）发送请求和接收结果，处理完成，子进程退出。

nging配置cgi比较繁琐，可以通过nc来[模拟一些这个流程](http://qjw.qiujinwu.com/blog/2013/12/22/nc_server)
``` cpp
#include <stdio.h>
int main()
{
    char buf_[1024];
    do{
        if(gets(buf_))
        {
            printf("recv : %s\n",buf_);
            fflush(stdout);
        }
    }while(1);
    return 0;
}
```
server
``` bash
king@king:/tmp$ gcc a.c
king@king:/tmp$ rm -f /tmp/f; mkfifo /tmp/f
king@king:/tmp$ cat /tmp/f | ./a.out | nc -l -p 12345 127.0.0.1 > /tmp/f
```
client
``` bash
king@king:/tmp$ nc 127.0.0.1 12345
1
recv : 1
2
recv : 2
3
recv : 3
4
recv : 4
5
recv : 5
6
recv : 6
```

CGI的设计可以支持各种不同语言的程序，问题是性能太差，所以又搞出了什么fastcgi。

FastCGI，顾名思义为更快的 CGI，它允许在一个进程内处理多个请求，而不是一个请求处理完毕就直接结束进程，性能上有了很大的提高。这其中又以Php的[PHP-FPM](https://php-fpm.org/)比较典型，和Python的网关服务器将Http协议处理部分自行处理不同，Php网关服务器通常配合第三方Http服务器使用，例如[nginx](https://nginx.org/en/)或者[lighttpd](https://www.lighttpd.net/)。

[PHP-FPM](https://zhuanlan.zhihu.com/p/20694204)作为一个独立的网关服务器存在，和nginx类似，会有一个master进程接收请求（nginx的master进程做的事情少得多）和若干worker进程，每个worker进程会跑一个PHP解释器，是 PHP 代码真正执行的地方。master进程则监听端口，并接收来自Http服务器的请求，通常情况下使用unixsocket。

nginx作为一个通用的服务器，为了配合[PHP-FPM](https://php-fpm.org/)，需要使用插件[fastcgi_param](https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/)，根据配置该插件会作一些转换并请求代理到FPM的地址。

处理好了Http请求分析，调度传递之后，还有很多事情需要处理，如前面提到的【检查参数合法性】、【检查权限/访问控制】等，这就是web框架的事情了，web框架是为了提高开发效率，让开发者更聚焦于业务逻辑，直接裸写也是OK的。


