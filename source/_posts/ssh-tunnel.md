---
title: SSH隧道
date: 2017-12-06 19:07:25
tags:
- ssh
categories:
- 运营运维
- 代理
---

ssh常用来做远程ssl登录，通过添加公钥到`~/.ssh/authorized_keys`还可以实现`无密登录`

# 其他小tip
## 直接执行命令
``` bash
king@DESKTOP-89R692H:/home/king$ ssh king@192.168.1.2 pwd
king@192.168.1.2's password:
/home/king
```

## 拷贝文件
``` bash
king@DESKTOP-89R692H:/home/king$ scp test.sh king@192.168.1.2:/tmp
king@192.168.1.2's password:
test.sh                100%  124     0.1KB/s   00:00
king@DESKTOP-89R692H:/home/king$ scp king@192.168.1.2:/tmp/test.sh ./test.sh
king@192.168.1.2's password:
test.sh                100%  124     0.1KB/s   00:00
```

## 密码参数
在不适用公钥无密登录的时候，密码必需手动输入，无法通过命令行直接带进去

对此，可以使用开源软件[putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)来支持，后者有发布Windows版本，不过可以直接在Linux下编译

具体用法参考<http://qjw.qiujinwu.com/blog/2013/01/28/putty>

# 本地端口转发
假定host1是本地主机，host2是远程主机。由于种种原因，这两台主机之间无法连通。但是，另外还有一台host3，可以同时连通前面两台主机。因此，很自然的想法就是，通过host3，将host1连上host2。

``` bash
本地端口:目标主机:目标主机端口 -p 跳板机端口 跳板机username@跳板机host
```

端口转发的一个类似场景如nginx

> 只留一个对外的主机，并且开放80端口，通过nginx根据ServerName/Path来分发到upsteam(本机/其他主机的端口)，

## docker
默认情况docker的端口对于host是不可访问的，当然`-p`可以将端口share出来

这里模拟一种场景，测试服务器只开放了22/80端口，想通过ssh隧道访问其他端口

准备文件
``` bash
king@king:~/tmp/docker$ ls
authorized_keys  dockerfile  sources.list
```

`authorized_keys`存host的公钥，文件来自`~/.ssh/id_rsa.pub`,`sources.list`设置（比如阿里云）的源，这里我们用它来实现无密登录docker的ssh

dockerfile
``` dockerfile
FROM ubuntu:16.04

MAINTAINER qiujinwu@gmail.com

RUN mkdir -p /app
WORKDIR /app

COPY sources.list /etc/apt

RUN apt-get update \
	&& apt-get install openssh-server netcat iputils-ping telnet net-tools -y \
	&& apt-get clean -y \
	&& apt-get autoclean

RUN mkdir /var/run/sshd
RUN echo 'root:1' |chpasswd
RUN mkdir /root/.ssh/
COPY authorized_keys /root/.ssh/

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

build docker镜像
``` bash
docker build -t sshd:0.1 ~/tmp/docker/
```
运行
``` bash
# 进入交互shell，再手动启动sshd
docker run -it --rm -p 10022:22 --name sshd sshd:0.1 /bin/bash
# 自动启动sshd
docker run -it --rm -p 10022:22 --name sshd sshd:0.1
# 关闭(另起shell）
docker stop sshd
```

### echo 服务器
``` bash
rm -f /tmp/f; mkfifo /tmp/f
cat /tmp/f | nc -l -p 40001 127.0.0.1 > /tmp/f
```

到此，我们可以启动一个进入shell的docker镜像，然后`/etc/init.d/sshd start`启动sshd，最后在shell中启动echo服务器

``` bash
king@king:~$ docker run -it --rm -p 10022:22 --name sshd sshd:0.1 /bin/bash
root@2f661b626247:/app# /etc/init.d/ssh start
 * Starting OpenBSD Secure Shell server sshd [ OK ]
root@2f661b626247:/app# rm -f /tmp/f; mkfifo /tmp/f
root@2f661b626247:/app# cat /tmp/f | nc -l -p 40001 127.0.0.1 > /tmp/f
root@2f661b626247:/app# 
```


## 端口转发
``` bash
# -N参数，表示只连接远程主机，不打开远程shell
# -T参数，表示不为这个连接分配TTY
ssh -NT -L localhost:40001:localhost:40001 -p 10022 root@127.0.0.1
```

这是，本机的sshd会监听40001端口，并且转交到本机的10022端口（docker的22端口），然后docker会交给自己的40001端口

接下来在本机运行客户端
``` bash
king@king:~$ nc 127.0.0.1 40001
aa
aa
bbb
bbb
```

## 跨主机
前面的例子第三方的跳板机收到请求后，转发到自己（localhost）的其他端口，也可以转发到其他主机

运行目标机docker
``` bash
king@king:~/.ssh$ docker run -it --rm --name sshd2 sshd:0.1 /bin/bash
root@bea2bbfe3625:/app# rm -f /tmp/f; mkfifo /tmp/f
# 这里不指定0.0.0.0
root@bea2bbfe3625:/app# cat /tmp/f | nc -l -p 40001 > /tmp/f
```

运行跳板机docker, 暴露端口22到10022，并且链接到目标机（使用别名up）
``` bash
docker run -it --rm -p 10022:22 --link=sshd2:up --name=sshd sshd:0.1
```

参考<http://qjw.qiujinwu.com/blog/2013/12/22/nc_server>


运行宿主机 ,将本机40001的数据转发到跳板机docker，然后再转发到名为up的docker的40001端口
``` bash
ssh -NT -L localhost:40001:up:40001 -p 10022 root@127.0.0.1
```

# 远程端口转发
host1与host2之间无法连通，必须借助host3转发。但是，特殊情况出现了，host3是一台内网机器，它可以连接外网的host1，但是反过来就不行，外网的host1连不上内网的host3。这时，"本地端口转发"就不能用了，怎么办？
解决办法是，既然host3可以连host1，那么就从host3上建立与host1的SSH连接，然后在host1上使用这条连接就可以了。

``` bash
外网端口:目标主机:目标主机端口 -p 外网ssh端口 外网机username@外网host
```

继续使用docker模拟

默认的bridge模式, host机会有一个x.x.x.1的ip和docker机器联通

运行目标机器
``` bash
king@king:~/.ssh$ docker run -it --rm --name sshd2 sshd:0.1 /bin/bash
root@556f737ec232:/app# rm -f /tmp/f; mkfifo /tmp/f
root@556f737ec232:/app# cat /tmp/f | nc -l -p 40001 > /tmp/f
```

运行跳板机，通知host机监听40001端口，并且转发给自己，然后自己再转发给up机器的40001端口
``` bash
king@king:~/tmp/docker$ docker run -it --rm --link=sshd2:up --name=sshd sshd:0.1 /bin/bash
root@fd1b60b1b84e:/app# ssh -NT -R 40001:up:40001 -p 22 king@172.17.0.1
king@172.17.0.1's password:
```

在host机执行测试程序即可
``` bash
king@king:~$ nc 127.0.0.1 40001
aa
aa
bbb
bbb
```

# SSL隧道
在本地绑定一个端口，发送往这个端口的数据都通过ssh到远程机器出去，本质上就是socks代理
``` bash
 ssh -NT -D 1080 king@192.168.1.219
```

浏览器可以安装个[switchyomega](https://github.com/FelisCatus/SwitchyOmega/releases)工具，然后设置socks5代理指向 localhost:1080，就可以让浏览器所有数据都从192.168.1.219出去，从而做些羞羞的事情


# 参考
1. <http://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html>
2. <http://qjw.qiujinwu.com/blog/2013/12/22/nc_server>
3. <http://www.cnblogs.com/wellbye/p/3163966.html>
