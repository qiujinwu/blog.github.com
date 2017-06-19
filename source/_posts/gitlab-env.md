---
title: 搭建gitlab测试环境
date: 2017-6-19 10:36:35
tags:
 - gitlab
categories:
 - 开发环境
---

# 准备docker
``` bash
mkdir -p /mnt/volumes/gitlab
cd /mnt/volumes/gitlab
mkdir config logs data
sudo docker run -it --rm  \
    --hostname localhost \
    --publish 8080:80 --publish 2222:22 \
    --name gitlab \
    --volume /mnt/volumes/gitlab/config:/etc/gitlab \
    --volume /mnt/volumes/gitlab/logs:/var/log/gitlab \
    --volume /mnt/volumes/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:latest
```

> 
--hostname
指定容器中绑定的域名，会在创建镜像仓库的时候使用到，这里绑定localhost
--publish
端口映射，冒号前面是宿主机端口，后面是容器expose出的端口
--volume
volume 映射，冒号前面是宿主机的一个文件路径，后面是容器中的文件路径

# 配置
启动之后就可以使用地址<http://127.0.0.1:8080>访问本地的gitlab，

启动之后需要设置初始的密码，以及注册新用户

完了创建group和project

**添加公钥，以便免密clone**，从~/.ssh/isa.pub文件中获取

然后就可以clone代码了

## 指定git clone 的端口
由于docker将22端口重定向到了主机的2222，所以需要修改**~/.ssh/config**文件，添加
``` ini
Host localhost
    Port 2222
```

# 参考
1. <http://www.jianshu.com/p/05e3bb375f64>
2. <https://stackoverflow.com/questions/3596260/git-remote-add-with-other-ssh-port>
