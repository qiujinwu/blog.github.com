---
title: Flask 处理websocket
date: 2017-03-16 19:51:19
tags: flask
categories:
 - 后端
---

最近要用flask弄一个websocket服务，由于ws的特殊性，若用正常的flask写法，就会**阻塞**当前线程，生产环境一般用[Gunicorn](http://gunicorn.org/)，没有测试过有无作异步处理，但是若用flask内置的容器，就会出问题。

之前学习Golang的时候，了解到[gorilla/websocket](https://github.com/gorilla/websocket)这个工程。在Go里面写ws逻辑很简单，直接在回调里面一个for循环就好，因为go运行时在底层会自动将goruntime的请求异步化，也就是用同步的代码写异步请求，而不是nodejs的那种回调满天飞的代码，很是赏心悦目。

Flask运行需要一个[WSGI 容器](http://docs.jinkan.org/docs/flask/deploying/wsgi-standalone.html)来提供服务，flask就内置了wsgi容器，不过性能一般，供测试足够了。

一些生产环境下的容器如
1. [Gunicorn](http://gunicorn.org/)
2. [Tornado](http://www.tornadoweb.org/en/stable/)
3. [Gevent](http://www.gevent.org/)

不同的容器使用的服务器模型不一样，比如

1. 多进程模型，来一请求就开一进程
2. 多线程模型，来一请求就开一线程
3. 进程池或者线程池，若池为空就排队，进程调度通常又通过外部的调度程序，例如nginx，或者直接用操作系统来调度（父进程bind，然后fork），多线程调度通常用一个队列来分配任务，或者利用操作系统来调度
4. 两者混合使用，有多个进程的池，每个进程有一个线程池

在大部分情况下，如果一个任务（请求）阻塞特别长时间，可能导致整个线程不可用，进而导致拒绝服务。在大部分**阻塞特别长**的情况下，后台只是简单地等待IO，这种情况下可以用全异步实现最大的并发。

写个例子
``` python
import time
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    time.sleep(1000)
    return 'Hello, World!'


@app.route('/a')
def hello_world1():
    return 'a'

if __name__ == '__main__':
    app.run(host="0.0.0.0",port=5000)
```

直接用python运行【python main.py】,访问<http://127.0.0.1:5000/a>成功，访问<http://127.0.0.1:5000/>在那等待，再访问<http://127.0.0.1:5000/a>也一并卡住了，这里可能只有一个线程在提供服务，而且sleep把这个线程阻塞掉了

换个容器

``` python
import time
from gevent import monkey
monkey.patch_all()
from flask import Flask
from gevent import wsgi
app = Flask(__name__)

@app.route('/')
def hello_world():
    time.sleep(1000)
    return 'Hello, World!'


@app.route('/a')
def hello_world1():
    return 'a'

if __name__ == '__main__':
    server = wsgi.WSGIServer(('0.0.0.0', 5000), app)
    server.serve_forever()
```

写个脚本，运行100个wget去请求<http://127.0.0.1:5000/>，然后再访问<http://127.0.0.1:5000/a>，结果**成功**。
``` bash
#!/bin/bash
for ((i=0;i<100;i++));do 
	(
	wget http://127.0.0.1:5000 
	)&
done
```