---
title: SSH远程自动执行交互任务
date: 2017-04-10 11:56:41
tags:
 - ssh
categories:
 - 开发环境
---

ssh远程到其他主机是比较安全和方便的办法，例如[**ssh king@1.2.3.4**]，若要自动执行某些命令（并立即返回结果，退出ssh），可以[**ssh king@1.2.3.4 ls**]

若要执行一些交互式的任务，比方登录一个mysql客户端，并且需要持续的使用（而不是简单地做个sql查询），那么上面的办法就不行了

``` bash
#!/bin/bash
ssh -t kingqiu@1.2.3.4 "mysql -h192.168.0.1 -uuser -ppassword --default-character-set=utf8mb4 test"
```

#### 参考
1. <https://superuser.com/questions/646196/ssh-and-execute-interactive-command>

