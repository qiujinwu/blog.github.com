---
title: 用docker构建c编译环境
date: 2017-04-12 22:45:32
tags:
 - docker
categories:
 - 开发环境
---


# Dockerfile
``` bash
king@king:~/tmp/dockerfile$ 
FROM ubuntu:16.04

MAINTAINER qiujinwu@gmail.com

RUN mkdir -p /app
WORKDIR /app

COPY sources.list /etc/apt
COPY run.sh /

RUN apt-get update \
	&& apt-get install build-essential -y \
	&& apt-get install gdb -y \
	&& apt-get clean -y \
	&& apt-get autoclean

ENTRYPOINT ["/run.sh"]
```

# Docker 入口脚本
用于支持gcc/g++/make命令，也可以直接进入shell
``` bash
#!/bin/bash
#for arg in "$@"
#do
#	echo "arg: ${arg}"
#done

declare -r g_red="\e[0;31;44m"
declare -r g_green="\e[0;32m"
declare -r g_blue="\e[0;34m"
declare -r g_end="\e[0m"

_err(){
	echo -e "${g_red}${@}${g_end}"
	return 1
}

do_make(){
	make "${@}"
}

do_gcc(){
	gcc "${@}"
}

do_gplus(){
	g++ "${@}"
}

if [ "${#}" -lt 1 ]
then
	/bin/bash
elif [ "${1}" == "make" -o "${1}" == "m" ]
then
	shift 1
	do_make "${@}"
elif [ "${1}" == "gcc" -o "${1}" == "g" ]
then
	shift 1
	do_gcc "${@}"
elif [ "${1}" == "g++" -o "${1}" == "+" ]
then
	shift 1
	do_gplus "${@}"
else
	_err "unsupport command"
fi
```

# apt源
``` bash
king@king:~/tmp/dockerfile$ cat sources.list 

deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
```

# 编译和测试
``` bash
#/bin/bash
# 编译镜像
docker build -t king:0.1 ~/tmp/dockerfile/
# 检查是否创建成功
docker images | grep king
# 运行
docker run -it --rm -v "$PWD":/app  king:0.1 gcc main.c -v
```

## 运行
``` bash
#!/bin/bash
docker run -it --rm -v "$PWD":/app  king:0.1 "${@}"
```

运行,**必须在当前目录运行**
``` bash
king@king:~/tmp/dockerfile$ ./b.sh g main.c 
king@king:~/tmp/dockerfile$ ./b.sh g main.c -g2
king@king:~/tmp/dockerfile$ ./b.sh 
root@c780882cbd3f:/app# gdb a.out 
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.04) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
```
