---
title: 同一台电脑多github帐号的ssh配置
date: 2017-02-13 16:20:05
tags:
 - github
categories:
 - 开发环境
---

为了免去push时输入密码，用git仓库时，通常都在本地生成ssh密钥对，并且配置到githu，bitbucket、gitlab等平台，例如[GitHub创建SSH Keys](http://blog.csdn.net/itmyhome1990/article/details/39668349)

``` bash
ssh-keygen -t rsa -C "youremail@xxx.com"
```

如果一切顺利,可以在用户目录里找到.ssh目录 里面有id_rsa和id_rsa.pub两个文件。这两个就是SSH Key的秘钥,id_rsa是私钥,id_rsa.pub是公钥。将id_rsa.pub内容拷贝到目标网站的某个地方几个。

多个github帐号不能用同一个key，所以必须创建两个。同样使用上面的命令，不过不能一直回车，存储文件时选择一个不同的路径，仍然确保在**.ssh**目录下。

新建名为**config**的文件，内容
```
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa

Host xxx.github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa1
```

HostName随意，后续需要用到，所以简单明了会比较好。

如果本地已经有仓库，或者直接clone下来的仓库，需要修改文件**.git/config**
```
[remote "origin"]
    url = git@xxx.github.com:king/blog.git
```

或者直接修改remote
``` bash
git remote rm origin
git remote add origin git@xxx.github.com:king/blog.git
```

或者clone时使用自定义的地址
``` bash
git clone git@xxx.github.com:king/blog.git
```

## 参考
1. <http://blog.csdn.net/itmyhome1990/article/details/42643233>
2. <https://gist.github.com/suziewong/4378434>
