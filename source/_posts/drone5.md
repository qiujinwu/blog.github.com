---
title: Drone学习笔记5
date: 2017-3-8 16:04:32
tags:
 - drone
 - golang
 - 持续集成
categories:
 - 源码学习
---

# 插件
drone不仅仅只是自动编译，还可以利用插件做一些其他事情，比如结果通知，打包目标文件成一个docker镜像。drone的插件可浏览<http://plugins.drone.io/>，一些常用的比如

1. <http://plugins.drone.io/drone-plugins/drone-docker/>
2. <http://plugins.drone.io/appleboy/drone-scp/>
3. <http://plugins.drone.io/drone-plugins/drone-slack/>
4. <http://plugins.drone.io/peloton/drone-rancher/>

插件的.drone.yml有自己的语法，具体要根据插件的帮助文档分析

# 矩阵编译
<http://readme.drone.io/usage/matrix-guide/> 意思就是根据几个变量的组合，基于所有的条件各执行一次pipeline

# 开发插件
官网提供了简单的插件教程，支持golang，bash，node和python
1. <http://readme.drone.io/plugins/creating-custom-plugins-bash/>
2. <http://readme.drone.io/plugins/creating-custom-plugins-golang/>

目前企业微信开发了API，理论上开发一个通知企业微信的插件也是可以的。

# 私有Docker Registry

随着docker越来越成熟，为了更好地管理这些docker，就冒出了**[docker compose](https://docs.docker.com/compose/)**这样的工具，以及更高层次的比如**[rancher](http://rancher.com/)**

而运行公司业务的docker镜像，显然也有必要放在自己的私有Registry内，然后通过drone插件一键上传到私有的Registry，当然也可以直接发布到rancher的测试环境。

搭建私有的Registry很简单，官方的registry就包含镜像【registry:2.0】，直接运行即可。
``` bash
docker run -p 5000:5000 registry:2.0
```

推送也很简单
``` bash
docker push localhost:5000/hello:latest
```

参考<http://dockone.io/article/324>
