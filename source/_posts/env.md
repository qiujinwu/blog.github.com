---
title: 一些环境的配置
date: 2017-03-05 15:14:24
tags:
 - golang
 - node
categories:
 - 开发环境
---


大部分情况下（特别是生产环境）最好是直接下载docker运行，简单可靠。偶尔为了方便本地测试，直接装在本地也有一定的好处，至少没那么折腾。

比较稳定（更新没那么频繁）的东西，例如nginx，mysql,直接apt-get就行了

# JDK
jdk默认情况下只有openjdk或者比较老的官方jdk，要安装最新的jdk 1.8，可以

添加jdk的ppa，参考<http://tipsonubuntu.com/2016/07/31/install-oracle-java-8-9-ubuntu-16-04-linux-mint-18/>

大部分ppa都需要翻墙，这很坑爹，也可以下载tar.gz包解压并配置环境变量，参考<http://blog.csdn.net/chenruicsdn/article/details/53583950>

``` bash
sudo mkdir -p /usr/lib/jvm
sudo tar -zxvf Downloads/jdk-8u112-Linux-x64.tar.gz -C /usr/lib/jvm
export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_112
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JAVA_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

# redis
默认的redis比较老，而有些新特性无法满足其他依赖需要，比如[sentry](https://sentry.io/welcome/)。

这里使用ppa<http://blog.csdn.net/chenlix/article/details/46696165>

``` bash
sudo apt-get install -y python-software-properties
sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:rwky/redis
sudo apt-get update
sudo apt-get install -y redis-server
```

关于sentry，可以docker<https://hub.docker.com/_/sentry/>

# Golang
去[官网](https://golang.org/dl/)下载最新的包，例如**go1.7.5.linux-amd64.tar.gz**，解压

``` bash
export GOROOT=/usr/lib/go/
export GOPATH=/home/abc/code/go
export PATH=$GOROOT/bin:$PATH
```

# Nodejs
去[官网](https://nodejs.org/en/download/)下载最新的包，例如**node-v6.10.0-linux-x64.tar.xz**，解压

``` bash
export NODE_HOME=/usr/lib/node
export PATH=$PATH:$NODE_HOME/bin
export NODE_PATH=$NODE_HOME/lib/node_modules
```

# Docker
直接安装[官网](https://docs.docker.com/engine/installation/linux/ubuntu/#install-using-the-repository)的指引安装docker-ce，还是很老旧，可以在apt-get purge卸载已经安装的docker版本，然后从<https://apt.dockerproject.org/repo/pool/main/d/docker-engine/>下载（例如）[docker-engine_1.13.1-0~ubuntu-xenial_amd64.deb](https://apt.dockerproject.org/repo/pool/main/d/docker-engine/docker-engine_1.13.1-0~ubuntu-xenial_amd64.deb)，然后dpkg -i 安装。

## 配置阿里云加速
参考<http://warjiang.github.io/devcat/2016/11/28/%E4%BD%BF%E7%94%A8%E9%98%BF%E9%87%8C%E4%BA%91Docker%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F/>

``` bash
//如果您的系统是 Ubuntu 15.04 16.04，Docker 1.9 以上

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/mirror.conf <<-'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/docker daemon -H fd:// --registry-mirror=https://2h3po24q.mirror.aliyuncs.com
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

注意，第一个ExecStart=不能省略，参见<http://askubuntu.com/questions/19320/how-to-enable-or-disable-services>

# Python
一般安装其他软件就把Python连带安装进来了，2和3都有，默认是Python2

``` bash
sudo update-alternatives --install /usr/bin/python python /usr/bin/python2 100
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3 500

python --version
Python 3.5.2

```

# rcconf
可以用rcconf启用/禁用某些服务的自启动

``` bash
sudo apt-get install rcconf
```


