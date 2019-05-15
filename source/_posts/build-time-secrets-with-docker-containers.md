---
title: Docker编译期间传递隐私数据
date: 2018-05-23 19:34:09
tags:
  - docker
categories:
  - 环境搭建
---

所谓的隐私数据，比如一些私有的token、密码、ssh key之类的不可公开但是编译期间却需要的信息

比如在docker编译期间，需要clone一个私有git仓库的代码，一般使用ssh key或者一个access token，这个密钥仅仅在编译期有用，并且不能暴露出去

# 几种常见思路
1. Dockerfile ENV variable
2. Docker build time variables

留意`build-arg`

``` bash
docker build –build-arg HTTP_PROXY=http://10.20.30.2:1234 .
```

这两种方式，实际都会在镜像中留下记录，也就是若获取到镜像，可以找到这些token，所以能达到目的，但并不安全

# [dockito/vault](https://github.com/dockito/vault)
[dockito/vault](https://github.com/dockito/vault)的思路很简单，在同一个docker环境下运行一个镜像，在里面运行一个http server。打包的docker镜像，则通过http协议去下载ssh key或者token，用完再删除。

这里的关键问题是`如何找到服务器的地址`

考虑到默认配置下，同一个docker环境下的容器共享同一个ip内网，所以在目标容器中获取默认网关IP，同时`dockito/vault`绑定到这个IP即可实现

## 测试
先打一个基础镜像，避免每次测试都要去apt-get和curl

file:`dockerfile.base`,你可以使用其他镜像，比如[Alpine Linux](https://alpinelinux.org/)
``` dockerfile
FROM ubuntu:14.04

RUN apt-get update && apt-get install -y curl git && \
        curl -L https://raw.githubusercontent.com/dockito/vault/master/ONVAULT > /usr/bin/ONVAULT && \
        chmod +x /usr/bin/ONVAULT
```
``` bash
docker build -t base:0.1 . -f Dockerfile.base
```

### 运行vault
``` bash
king@king:~/$ docker run --rm -it  -p 172.17.0.1:14242:3000 -v ~/.ssh:/vault/.ssh dockito/vault

> dockito-vault@1.0.0 start /usr/src/app
> node index.js

Service started on port 3000
```
或者调试模式
``` bash
king@king:~/$ docker run --rm -it  -p 172.17.0.1:14242:3000 -v ~/.ssh:/vault/.ssh dockito/vault /bin/sh
/usr/src/app # node index
Service started on port 3000
```

在宿主主机访问下面地址确认，参见<https://github.com/dockito/vault/blob/master/index.js>

1. <http://172.17.0.1:14242/_ping>
2. <http://172.17.0.1:14242/ONVAULT>
3. <http://172.17.0.1:14242/ssh.tgz>

### 目标镜像
``` dockerfile
FROM base:0.1
RUN ONVAULT git clone git@gitlab.kimq.cn:kingqiu/flask-swagger.git
```
``` bash
king@king:~/$ docker build -t b:0.1 .
Sending build context to Docker daemon   131 MB
Step 1/2 : FROM base:0.1
 ---> e8e50a0502d0
Step 2/2 : RUN ONVAULT git clone git@gitlab.kimq.cn:kingqiu/flask-swagger.git
 ---> Running in 2747e4ef93b9
[Dockito Vault] Downloading private keys...
[Dockito Vault] Using ssh key: id_rsa
[Dockito Vault] Executing command: git clone git@gitlab.kimq.cn:kingqiu/flask-swagger.git
Cloning into 'flask-swagger'...
[Dockito Vault] Removing private keys...
 ---> 3591a1e64c2d
Removing intermediate container 2747e4ef93b9
Successfully built 3591a1e64c2d
```

具体怎么获取ssh key以及销毁，在脚本<https://github.com/dockito/vault/blob/master/ONVAULT>

## 使用定制ssh key

生成key，名称务必保持`id_rsa`
``` bash
(venv) king@king:~/$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/king/.ssh/id_rsa):  id_rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in id_rsa.
Your public key has been saved in id_rsa.pub.
The key fingerprint is:
SHA256:fXnNp2bZXa9G7EhrjqsaZS/4W9XKgGB+YN5esolUWnQ king@king
The key's randomart image is:
+---[RSA 2048]----+
|        . E      |
|       . .       |
|      = o        |
|     = B o   o o |
|      = S + +.o =|
|     . B * =.oo+=|
|      + = o.o== +|
|       o o .+oo. |
|      ..+oo+...  |
+----[SHA256]-----+
```

新建`config`，确保和上面步骤的id_rsa同一目录，假如目录是`/home/king/tmp`，留意选项`StrictHostKeyChecking`
``` bash
Host *
    StrictHostKeyChecking no
```

运行server
``` bash
docker run --rm -it -p 172.17.0.1:14242:3000 -v ~/tmp:/vault/.ssh dockito/vault
```

将id_rsa.pub作为deploy key设置到gitlab/github上

然后重新build

### 非`id_rsa`的key

在脚本[ONVAULT](https://github.com/dockito/vault/blob/master/ONVAULT)，若设置了环境变量`VAULT_SSH_KEY`，会自动更新配置config
``` bash
if [[  "$VAULT_SSH_KEY" != "id_rsa" ]]; then
	# configure the ssh to any host to use this ssh key
	echo -e "\nHost *\nIdentityFile ~/.ssh/$VAULT_SSH_KEY" >> ~/.ssh/config
fi
```

重命名

``` bash
mv id_rsa gitlab && mv id_rsa.pub gitlab.pub
```

留意环境变量`VAULT_SSH_KEY`

``` dockerfile
FROM base:0.1
ENV VAULT_SSH_KEY gitlab
RUN ONVAULT git clone git@gitlab.kimq.cn:kingqiu/flask-swagger.git
```
``` bash
king@king:~/$ ./b.sh 
Untagged: b:0.1
Deleted: sha256:60987f8c94
Sending build context to Docker daemon   131 MB
Step 1/3 : FROM base:0.1
 ---> e8e50a0502d0
Step 2/3 : ENV VAULT_SSH_KEY gitlab
 ---> Using cache
 ---> 8b92fa1adab3
Step 3/3 : RUN ONVAULT git clone git@gitlab.kimq.cn:kingqiu/flask-swagger.git
 ---> Running in 43e70f25b64e
[Dockito Vault] Downloading private keys...
[Dockito Vault] Using ssh key: gitlab
[Dockito Vault] Executing command: git clone git@gitlab.kimq.cn:kingqiu/flask-swagger.git
Cloning into 'flask-swagger'...
Warning: Permanently added 'gitlab.kimq.cn,139.224.170.165' (ECDSA) to the list of known hosts.
[Dockito Vault] Removing private keys...
 ---> 1400a685ef0a
Removing intermediate container 43e70f25b64e
Successfully built 1400a685ef0a
```

# 问题
``` bash
[Dockito Vault] Executing command: git clone git@gitlab.kimq.cn:kingqiu/flask-swagger.git
Cloning into 'flask-swagger'...
fatal: cannot run ssh: No such file or directory
fatal: unable to fork
```
> apk add --update --virtual openssh-client

``` bash
Collecting git+ssh://git@gitlab.kimq.cn/kingqiu/flask-swagger.git (from -r requirements.txt (line 57))
  Cloning ssh://git@gitlab.kimq.cn/kingqiu/flask-swagger.git to /tmp/pip-o1noeq66-build
Bad owner or permissions on /root/.ssh/config
fatal: Could not read from remote repository.
```
> chmod 0600 config && chown `whoami`:`id -g -n` config

# 参考
1. <https://elasticcompute.io/2016/01/22/build-time-secrets-with-docker-containers/>
2. <https://github.com/dockito/vault>
3. <https://github.com/mdsol/docker-ssh-exec>
