---
title: Celery适配RocketMQ
date: 2020-02-05 00:00:00
tags:
 - celery
 - rocketmq
categories:
 - Celery
---

# RocketMQ测试
## 安装
``` bash
$ git clone git@github.com:apache/rocketmq-docker.git
$ sh stage.sh 4.5.0
$ cd stages/4.5.0/templates
$ ./play-docker.sh alpine
$ $ docker ps
CONTAINER ID IMAGE                                  COMMAND        PORTS         NAMES
c56d46fead7b apacherocketmq/rocketmq:4.5.0-alpine   "sh mqbroker"  0.0.0.......  rmqbroker
94caf06791b1 apacherocketmq/rocketmq:4.5.0-alpine   "sh mqnamesrv" 10909/tcp...  rmqnamesrv
```

> 可以参考<https://github.com/apache/rocketmq-docker#how-to-verify-rocketmq-works-well> 确认下是否OK

## 客户端

rocketmq-client-python is a lightweight wrapper around rocketmq-client-cpp, so you need install librocketmq first.

### ClientCPP
``` bash
$ git@github.com:apache/rocketmq-client-python.git
$ cd rocketmq-client-cpp
$ ./build.sh
$ ls bin/libro*
bin/librocketmq.a  bin/librocketmq.so
$ sudo cp bin/librocketmq.* /usr/lib/x86_64-linux-gnu/
```

> 安装过程中会wget一些第三方库,比如boost, 速度呵呵呵, 最好搞个梯子, 设置`$HTTP_PROXY`和`$HTTPS_PROXY`, 另外检查下输出, 可能还有一些其他库, 基本都能`apt install **-dev`

### Sample
``` bash
$ pip install rocketmq-client-python==2.0.0
```

不要`pip install rocketmq`, 此库已废

> This project is currently being upstreamed to apache/rocketmq-client-python

生产者
``` python
from rocketmq.client import Producer, Message

producer = Producer('PID-XXX')
producer.set_name_server_address('127.0.0.1:9876')
producer.start()

msg = Message('YOUR-TOPIC')
msg.set_keys('XXX')
msg.set_tags('XXX')
msg.set_body('XXXX')
ret = producer.send_sync(msg)
print(ret.status, ret.msg_id, ret.offset)
producer.shutdown()
```
消费者
``` python
import time

from rocketmq.client import PushConsumer, ConsumeStatus


def callback(msg):
    print(msg.id, msg.body)
    return ConsumeStatus.CONSUME_SUCCESS


consumer = PushConsumer('CID_XXX')
consumer.set_name_server_address('127.0.0.1:9876')
consumer.subscribe('YOUR-TOPIC', callback)
consumer.start()

while True:
    time.sleep(3600)

consumer.shutdown()
```

# Celery
``` bash
pip install redis==2.10.6
pip install celery==4.0.2
```

> 不同的redis有兼容性问题, 参考<https://blog.csdn.net/weixin_42260750/article/details/84370991>

``` python
from __future__ import absolute_import, unicode_literals
import sys
from celery import Celery

app = Celery('project', broker='redis://localhost:6379/0')

@app.task
def add(x, y):
    return x + y

@app.task
def test():
    print("done")
    return 0

if __name__ == "__main__":
    if len(sys.argv) > 1:
        result = add.delay(1,2)
        exit(0)
    argv = [
        'worker',
        '--loglevel=DEBUG',
    ]
    app.worker_main(argv)
```

``` bash
# 运行worker
$ python main.py
# 触发task
$ python main.py task
```



# 参考
1. [Apache RocketMQ python client](https://github.com/apache/rocketmq-client-python)
2. [Apache RocketMQ Docker](https://github.com/apache/rocketmq-docker#b-stage-a-specific-version)
3. [Apache RocketMQ cpp client](https://github.com/apache/rocketmq-client-cpp/tree/master#build-and-install)
4. [Celery 初步](http://docs.jinkan.org/docs/celery/getting-started/first-steps-with-celery.html)
