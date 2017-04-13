---
title: 用docker构建c编译环境2
date: 2017-04-13 09:32:17
tags:
 - docker
categories:
 - 开发环境
---

可以编译了，最终需要运行，而且很有必要编译和运行是同一个环境

下面以一个libevent程序来说明问题

main.c
``` c
#include <sys/types.h>  
#include <sys/stat.h>  
#include <time.h>  
#ifdef _EVENT_HAVE_SYS_TIME_H  
#include <sys/time.h>  
#endif  
#include <stdio.h>  
#include <event2/event.h>  
#include <event2/event_struct.h>  
#include <event2/util.h>  
  
struct timeval lasttime;  
int event_is_persistent;  
  
static void timeout_cb(evutil_socket_t fd, short event, void *arg)  
{  
	printf("timeout\n");
}  
  
int main(int argc, char **argv)  
{  
    struct event timeout;       //创建事件  
    struct timeval tv;  
    struct event_base *base;    //创建事件"总管"的指针  
    int flags;                      
    //事件标志,超时事件可不设EV_TIMEOUT,因为在添加事件时可设置  
  
    event_is_persistent = 1;  
    flags = EV_PERSIST;  
    /* Initalize the event library */  
    base = event_base_new();            //创建事件"总管"  
    /* Initalize one event */  
    event_assign(&timeout, base, -1, flags, timeout_cb, (void*) &timeout);  
    evutil_timerclear(&tv);  
    tv.tv_sec = 1;  
    event_add(&timeout, &tv);       //添加事件,同时设置超时时间  
    evutil_gettimeofday(&lasttime, NULL);  
    event_base_dispatch(base);      //循环监视事件,事件标志的条件发生,就调用回调函数  
    return (0);  
}
```

Dockerfile
```
FROM king_release:0.1

MAINTAINER qiujinwu@gmail.com

RUN mkdir -p /app
WORKDIR /app

COPY sources.list /etc/apt
COPY run.sh /

RUN apt-get update \
	&& apt-get install build-essential -y \
	&& apt-get install gdb -y \
	&& apt-get install libevent-dev -y \
	&& apt-get clean -y \
	&& apt-get autoclean

ENTRYPOINT ["/run.sh"]
```

Dockerfile2
```
FROM ubuntu:16.04

MAINTAINER qiujinwu@gmail.com

COPY sources.list /etc/apt

RUN apt-get update \
	&& apt-get install libevent-2.0-5 -y \
	&& apt-get clean -y \
	&& apt-get autoclean
```

Dockerfile3
```
FROM king_release:0.1
MAINTAINER qiujinwu@gmail.com
ADD a.out /
CMD ["/a.out"]
```

``` bash
# 编译运行时基础镜像
docker build -f $PWD/Dockerfile2 -t king_release:0.1 ~/tmp/dockerfile/
# 编译 打包编译服务镜像
docker build -t king_build:0.1 ~/tmp/dockerfile/
# 编译程序
./b.sh + main.c -levent
# 编译最终运行的镜像
docker build -f $PWD/Dockerfile3 -t a.out:0.1 ~/tmp/dockerfile/
```

编译后如下
```
king@king:~/tmp/dockerfile$ docker images
REPOSITORY      TAG                 IMAGE ID            CREATED             SIZE
a.out           0.1                 bd79dd4a7add        2 minutes ago       172 MB
king_build      0.1                 e0d3c8b3e691        23 minutes ago      479 MB
king_release    0.1                 cc28cd7564c0        33 minutes ago      172 MB
```

运行程序
``` bash
king@king:~/tmp/dockerfile$ docker run -it --rm a.out:0.1
timeout
timeout
timeout
timeout
```

停止容器和删除镜像，*无需删除容器*
``` bash
# 停止镜像
docker ps | grep a.out | awk '{print $1}' | xargs -i docker stop {}
# 删除镜像
docker images | grep a.out | awk '{print $3}' | xargs -i docker rmi {}
```


# 裁剪
上面的运行时还有100多M，还是挺大的，利用静态编译可以压缩到更小。
```
king@king:~/tmp/dockerfile$ ./b.sh + main.c -levent
king@king:~/tmp/dockerfile$ ./b.sh + main.c -levent -static -o b.out
king@king:~/tmp/dockerfile$ ll
总用量 1148
-rwxr-xr-x  1 root root    9024 4月  13 09:48 a.out*
-rwxr-xr-x  1 root root 1112800 4月  13 09:49 b.out*
```

可以看到，静态连接的可执行程序大了非常多，但是他没有那些系统库/三方库的依赖，所以可以很方便的使用精简版的docker来运行

Dockerfile4
```
FROM scratch
ADD b.out /
CMD ["/b.out"]
```

``` bash
docker build -f $PWD/Dockerfile4 -t b.out:0.1 ~/tmp/dockerfile/
```

``` bash
king@king:~/tmp/dockerfile$ docker images
REPOSITORY      TAG                 IMAGE ID            CREATED             SIZE
b.out           0.1                 1176fc755f3d        2 minutes ago       1.11 MB
a.out           0.1                 bd79dd4a7add        32 minutes ago      172 MB
king_build      0.1                 e0d3c8b3e691        53 minutes ago      479 MB
king_release    0.1                 cc28cd7564c0        About an hour ago   172 MB
```

可以看到打出来的docker镜像非常非常小

``` bash
king@king:~/tmp/dockerfile$ docker run -it --rm b.out:0.1
timeout
timeout
timeout
timeout
```

``` bash
(venv) king@king:~/tmp/dockerfile$ ldd a.out 
	linux-vdso.so.1 =>  (0x00007ffdd8157000)
	libevent-2.0.so.5 => /usr/lib/x86_64-linux-gnu/libevent-2.0.so.5 (0x00007f85e9e7d000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f85e9ab4000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f85e9896000)
	/lib64/ld-linux-x86-64.so.2 (0x000055b917112000)
(venv) king@king:~/tmp/dockerfile$ ldd b.out 
	不是动态可执行文件
(venv) king@king:~/tmp/dockerfile$ file b.out 
b.out: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, 
for GNU/Linux 2.6.32, BuildID[sha1]=eb066746936748bf4438aa6473cfcd101759dd95, not stripped
```


# 常见问题
1. 部分设备只安装了动态库，没有匹配的静态库，导致静态编译失败
2. 静态编译时存在一个顺序问题，如果在链接时，并未发现引用就不会链接任何符号，而右边的对象如果又依赖这些库，就会出现链接失败，如下


还是上面的代码main.c
``` bash
king@king:~/tmp/a$ gcc main.c -c
king@king:~/tmp/a$ gcc -levent -static main.o
main.o：在函数‘main’中：
main.c:(.text+0x5e)：对‘event_base_new’未定义的引用
main.c:(.text+0x9b)：对‘event_assign’未定义的引用
main.c:(.text+0xd8)：对‘event_add’未定义的引用
main.c:(.text+0xf6)：对‘event_base_dispatch’未定义的引用
collect2: error: ld returned 1 exit status
king@king:~/tmp/a$ gcc main.o -levent -static
```
