---
title: Sentry
date: 2017-03-10 09:43:34
tags:
 - sentry
categories:
 - 开发环境
---

# 安装Sentry
推荐用docker来安装，参考**<https://hub.docker.com/_/sentry/>**
``` bash
docker pull sentry
docker pull redis
docker pull postgres
```

运行redis
``` bash
docker run -d --name sentry-redis redis
```

运行progress
```
docker run -d --name sentry-postgres -e POSTGRES_PASSWORD=secret -e POSTGRES_USER=sentry postgres
```

使用sentry命令行生成一个密钥，用于在sentry容器之间授权
``` bash
king@king:~$ docker run --rm sentry config generate-secret-key
fc=k^xur0()hb7cqi81z@&b!d9cy6l_-j#&36)kk28u^%ldx*l
```

如果数据库是第一次使用（新的），用下面的命令创建表结构（初始化）
``` bash
docker run -it --rm -e SENTRY_SECRET_KEY='<secret-key>' \
	--link sentry-postgres:postgres --link sentry-redis:redis sentry upgrade
```

在这个过程中会提示创建用户
``` bash
Would you like to create a user account now? [Y/n]: Y
Email: email@gmail.com
Password:
Repeat for confirmation: 
Should this user be a superuser? [y/N]: Y
User created: email@gmail.com

```

运行服务器（Web）

注意**-p 8080:9000**
``` bash
docker run -d --name my-sentry -e SENTRY_SECRET_KEY='<secret-key>' \
	-p 8080:9000 \
	--link sentry-redis:redis --link sentry-postgres:postgres sentry
```

sentry还需要 celery beat 和 celery workers干活，所以再启动两个容器
``` bash
docker run -d --name sentry-cron -e SENTRY_SECRET_KEY='<secret-key>' \
	--link sentry-postgres:postgres --link sentry-redis:redis sentry run cron
docker run -d --name sentry-worker-1 -e SENTRY_SECRET_KEY='<secret-key>'  \
	--link sentry-postgres:postgres --link sentry-redis:redis sentry run worker
```


# Go Client
sentry client for go **<https://docs.sentry.io/clients/go/>**，只需要简单的封裝就可以适配gin
``` go
package main

import (
	"github.com/getsentry/raven-go"
	"net/http"
	"fmt"
	"github.com/gin-gonic/gin"
	"runtime/debug"
	"errors"
)

func Middleware(dsn string) gin.HandlerFunc {
	if dsn == "" {
		panic("Error: No DSN detected!\n")
	}
	raven.SetDSN(dsn)
	var client = raven.DefaultClient

	return func(c *gin.Context) {
		defer func() {
			if err := recover(); err != nil {

				flags := map[string]string{
					"endpoint": c.Request.RequestURI,
				}

				debug.PrintStack()
				rvalStr := fmt.Sprint(err)
				packet := raven.NewPacket(rvalStr, raven.NewException(
					errors.New(rvalStr),
					raven.NewStacktrace(2, 3, nil)),
				)
				client.Capture(packet, flags)
				c.Writer.WriteHeader(http.StatusInternalServerError)

				//const size = 1 << 12
				//buf := make([]byte, size)
				//n := runtime.Stack(buf, false)
				//client.CaptureMessage(fmt.Sprintf("%v\nStacktrace:\n%s", err, buf[:n]),nil)
				//c.Writer.WriteHeader(http.StatusInternalServerError)
			}
		}()
		c.Next()
	}
}
```

参考

1. <https://github.com/dz0ny/martini-sentry>
2. <https://github.com/gin-gonic/gin-sentry>

``` go
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/getsentry/raven-go"
	"errors"
	"fmt"
)

func main() {
	r := gin.Default()

	r.Use(Middleware("http://5138362834d22ecbd@localhost:8080/2"))
	// r.Use(gin.Recovery())

	r.GET("/ping", func(c *gin.Context) {
		raven.CaptureErrorAndWait(errors.New("this is a string error"), nil)
		panic(fmt.Sprintf("this is a panic"))
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run(":12345")
}
```

# Python client
``` python
# -*- coding: utf-8 -*-
from raven.contrib.flask import Sentry
from flask import Flask, session, redirect, url_for, escape, request
from flask_script import Manager

sentry = Sentry()

def create_app():
    app = Flask(__name__)
    sentry.init_app(app, dsn='http://5138362834d22ecbd@localhost:8080/2')
    return app

app = create_app()
sentry.captureMessage('hello, world!.shit');
```

# 不用docker
先准备redis，mysql或者postgres

## 安装sentry
apt-get 安装的依赖可能有所差异

``` bash
apt-get install libpq-dev libffi-dev python-mysqldb libmysqlclient-dev \
	libssl-dev libjpeg9 libjpeg9-dev libpng16-16 libpng3 libpng16-dev
pip install sentry pymysql MySQL-python
```

## 生成初始化配置
``` bash
(venv) king@kingqiu:~/proj$ sentry init .
(venv) king@kingqiu:~/proj$ ls
config.yml  sentry.conf.py
```

## 修改配置
### 数据库
默认是postgres
``` javascript
DATABASES = {
    'default': {
        'ENGINE': 'sentry.db.postgres',
        'NAME': 'sentry',
        'USER': 'postgres',
        'PASSWORD': '',
        'HOST': '',
        'PORT': '',
    }
}
```

改成mysql，其中的【NAME】表示数据库名，需要事先新建该数据库
``` javascript
DATABASES = {
    'default': {
        # You can swap out the engine for MySQL easily by changing this value
        # to ``django.db.backends.mysql`` or to PostgreSQL with
        # ``django.db.backends.postgresql_psycopg2``
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'sentry',
        'USER': 'username',
        'PASSWORD': 'password',
        'PORT': '3306',
        'HOST': 'localhost'
    }
}
```

### Redis
``` ini
# See https://docs.sentry.io/on-premise/server/queue/ for more
# information on configuring your queue broker and workers. Sentry relies
# on a Python framework called Celery to manage queues.

BROKER_URL = 'redis://localhost:6379'
```

### IP/端口
``` ini
SENTRY_WEB_HOST = '0.0.0.0'
SENTRY_WEB_PORT = 9000
SENTRY_WEB_OPTIONS = {
    # 'workers': 3,  # the number of web workers
    # 'protocol': 'uwsgi',  # Enable uwsgi protocol instead of http
}
```

## 运行
### 初始化
创建表神马的，config后面的.表示路径，自行修改。

在这个过程中，可以一次性创建超级用户
``` bash
sentry --config=. upgrade
```

### 创建新用户
``` bash
sentry --config=. createsuperuser
sentry --config=. createuser
```

### 启动
####  启动Web服务
``` bash
sentry --config=. start
```

####  启动Wokers
``` bash
sentry --config=. celery worker -B
```



