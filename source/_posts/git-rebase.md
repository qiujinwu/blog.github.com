---
title: git rebase 合并commit
date: 2017-04-17 16:13:05
tags:
 - git
categories:
 - 开发环境
---

对git基本上是记住一些常用的功能，其他用到了去查手册，这里备注一个我不常用但挺实用的功能

一句话，将若干commit合并成一个。

# 本地多个commit合并
``` bash
# 本地新建10个commit
for ((i=1;i<10;i++));do echo $i > a;git add .;git commit -m commit$i;done;
```

log如下
``` bash
* a69d8fb - (HEAD -> master) commit9 (2 分钟前) <King>
* 6b24d3a - commit8 (2 分钟前) <King>
* 98aeb92 - commit7 (2 分钟前) <King>
* 8b2120b - commit6 (2 分钟前) <King>
* 8660ff5 - commit5 (2 分钟前) <King>
* 38b1be3 - commit4 (2 分钟前) <King>
* 07be2fa - commit3 (2 分钟前) <King>
* 139aabd - commit2 (2 分钟前) <King>
* f570edb - commit1 (2 分钟前) <King>
```

``` bash
king@king:/tmp/git/work$ git rebase -i f570edb
# 将后面的提交的pick都改成s，保存确认commit的描述
pick 139aabd commit2
s 07be2fa commit3
s 38b1be3 commit4
s 8660ff5 commit5
s 8b2120b commit6
s 98aeb92 commit7
s 6b24d3a commit8
s a69d8fb commit9
king@king:/tmp/git/work$ git log
commit c040772af1dd96cd3311112193e39cbfd7a93e28
Author: King <qiujinwu456@gmail.com>
Date:   Mon Apr 17 16:17:11 2017 +0800
    commit2
    commit3
    commit4
    commit5
    commit6
    commit7
    commit8
    commit9
commit f570edbf5bbe3ba661c36916c36ac12227e1cdd1
Author: King <qiujinwu456@gmail.com>
Date:   Mon Apr 17 16:17:11 2017 +0800
    commit1
```

可以看到最新的9个已经merge成一个，但是第一个和第二个还是分开了（**搞不懂为啥要这么设计，将第一个pick改成s，或者其他操作都不行**）

如果一定要合并，可以参考<http://stackoverflow.com/questions/2563632/how-can-i-merge-two-commits-into-one>
``` bash
# 清掉第一条commit，但是保留代码
git reset --soft "HEAD^"
# 复用上一次的commit
git commit --amend
```

# 多人合作
新建三个目录
``` bash
king@king:/tmp/git$ tree . -d 2
.
├── origin
├── work
└── work1
```

``` bash
cd /tmp/git/work
git init .
for ((i=1;i<10;i++));do echo $i > a;git add .;git commit -m commit$i;done;
git remote add origin /tmp/git/origin/
cd ../origin
git init --bare
cd -
git push origin master
cd ../work1
git clone /tmp/git/origin/ .
```

到目前为止，work和work1共享了origin仓库

接下来让别人push一些修改到master，自己也做了一些commit
``` bash
cd /tmp/git/work
for ((i=1;i<10;i++));do echo $i > b;git add .;git commit -m local$i;done;
cd ../work1
git config user.name "another king"
for ((i=1;i<10;i++));do echo $i > c;git add .;git commit -m remote$i;done;
git push origin master
cd -
```

在work pull远程代码之后，log是这样的
```
git pull origin master
```

``` bash
*   3d5b7d2 - (HEAD -> master) Merge branch 'master' of /tmp/git/origin (19 秒钟前) <King>
|\  
| * c83b1e9 - (origin/master) remote9 (40 秒钟前) <another king>
| * 0c6e331 - remote8 (40 秒钟前) <another king>
| * 17a8fcd - remote7 (40 秒钟前) <another king>
| * c685ee7 - remote6 (40 秒钟前) <another king>
| * 8868d35 - remote5 (40 秒钟前) <another king>
| * 3b724d2 - remote4 (40 秒钟前) <another king>
| * d82b1bd - remote3 (40 秒钟前) <another king>
| * e9865f1 - remote2 (40 秒钟前) <another king>
| * 2c86a3b - remote1 (40 秒钟前) <another king>
* | 1384e0c - local9 (40 秒钟前) <King>
* | ae7c46e - local8 (40 秒钟前) <King>
* | bbea9bb - local7 (40 秒钟前) <King>
* | 88e2fbb - local6 (40 秒钟前) <King>
* | 60b7ac3 - local5 (40 秒钟前) <King>
* | c5494bd - local4 (40 秒钟前) <King>
* | 5dfe9a9 - local3 (40 秒钟前) <King>
* | 18ecbb4 - local2 (40 秒钟前) <King>
* | 76662a5 - local1 (40 秒钟前) <King>
|/  
* 8383510 - commit9 (48 秒钟前) <King>
* 0a15467 - commit8 (48 秒钟前) <King>
* 3f76d47 - commit7 (48 秒钟前) <King>
* 3d469ec - commit6 (48 秒钟前) <King>
* cdcf488 - commit5 (48 秒钟前) <King>
* cab099e - commit4 (48 秒钟前) <King>
* 77a35fe - commit3 (48 秒钟前) <King>
* 5b8b08e - commit2 (48 秒钟前) <King>
* f1d3575 - commit1 (49 秒钟前) <King>

```

这里有个问题，别人的改动插入到我的改动之中，若改成rebase，就不一样
``` bash
king@king:/tmp/git/work$ git pull --rebase origin master
remote: 对象计数中: 26, 完成.
remote: 压缩对象中: 100% (18/18), 完成.
remote: Total 26 (delta 0), reused 0 (delta 0)
展开对象中: 100% (26/26), 完成.
来自 /tmp/git/origin
 * branch            master     -> FETCH_HEAD
   4245282..ff5ceb9  master     -> origin/master
首先，回退分支以便在上面重放您的工作...
应用：local1
应用：local2
应用：local3
应用：local4
应用：local5
应用：local6
应用：local7
应用：local8
应用：local9
```

生成后的log如下
``` bash
* d6f1150 - (HEAD -> master) local9 (29 秒钟前) <King>
* 3ea2ae7 - local8 (29 秒钟前) <King>
* a1dcc9b - local7 (29 秒钟前) <King>
* 65488ec - local6 (29 秒钟前) <King>
* 417f852 - local5 (29 秒钟前) <King>
* e460dbd - local4 (29 秒钟前) <King>
* 7a0e9a5 - local3 (29 秒钟前) <King>
* cc58778 - local2 (29 秒钟前) <King>
* e69d861 - local1 (29 秒钟前) <King>
* ff5ceb9 - (origin/master) remote9 (54 秒钟前) <another king>
* dc8997b - remote8 (54 秒钟前) <another king>
* ef231f5 - remote7 (54 秒钟前) <another king>
* e7af01a - remote6 (54 秒钟前) <another king>
* 3ed0382 - remote5 (54 秒钟前) <another king>
* 1dce6c2 - remote4 (54 秒钟前) <another king>
* f726941 - remote3 (54 秒钟前) <another king>
* 13b51a6 - remote2 (54 秒钟前) <another king>
* 949d038 - remote1 (54 秒钟前) <another king>
* 4245282 - commit9 (60 秒钟前) <King>
* 6713be4 - commit8 (60 秒钟前) <King>
* d5c11c8 - commit7 (60 秒钟前) <King>
* 3e07d30 - commit6 (60 秒钟前) <King>
* 7265b3d - commit5 (60 秒钟前) <King>
* 0deb807 - commit4 (60 秒钟前) <King>
* 9b232a4 - commit3 (60 秒钟前) <King>
* bfd3ad1 - commit2 (60 秒钟前) <King>
* 108010e - commit1 (60 秒钟前) <King>
```

原理很简单
> --rebase,这里表示把你的本地当前分支里的每个提交(commit)取消掉，并且把它们临时 保存为补丁(patch)(这些补丁放到".git/rebase"目录中),然后把本地当前分支更新 为最新的"origin"分支，最后把保存的这些补丁应用到本地当前分支上。


参考<http://blog.csdn.net/hudashi/article/details/7664631/>

# 开发分支合并
大部分情况下，不会直接基于master分支，而是自己拉一个分支下来，然后commit，最后创建一个pull request。

``` bash
cd /tmp/git/work
git init .
for ((i=1;i<10;i++));do echo $i > a;git add .;git commit -m commit$i;done;
git remote add origin /tmp/git/origin/
cd ../origin
git init --bare
cd -
git push origin master
cd ../work1
git clone /tmp/git/origin/ .
```

到目前为止，work和work1共享了origin仓库

接下来让别人push一些修改到master，自己**开个新分支**也做了一些commit。最后切回master，并且拉取别人提交到master的commit
``` bash
cd /tmp/git/work
git checkout -b tmp
for ((i=1;i<10;i++));do echo $i > b;git add .;git commit -m local$i;done;
cd ../work1
git config user.name "another king"
for ((i=1;i<10;i++));do echo $i > c;git add .;git commit -m remote$i;done;
git push origin master
cd -
git checkout master
git pull origin master
```

一般情况是直接git merge master将别人的commit合并到临时分支
``` bash
git checkout tmp
git merge master
```

和之前的直接pull一样，别人的提交穿插在自己的commit中间
``` bash
*   36d5aa7 - (HEAD -> tmp) Merge branch 'master' into tmp (29 秒钟前) <King>
|\  
| * f5c0dc8 - (origin/master, master) remote9 (2 分钟前) <another king>
| * 45ec76f - remote8 (2 分钟前) <another king>
| * 095dc1b - remote7 (2 分钟前) <another king>
| * 7999947 - remote6 (2 分钟前) <another king>
| * 889f88c - remote5 (2 分钟前) <another king>
| * 303ab0c - remote4 (2 分钟前) <another king>
| * cff4cb8 - remote3 (2 分钟前) <another king>
| * c4995a0 - remote2 (2 分钟前) <another king>
| * 56bf566 - remote1 (2 分钟前) <another king>
* | 589f098 - local9 (2 分钟前) <King>
* | e575680 - local8 (2 分钟前) <King>
* | 2a14876 - local7 (2 分钟前) <King>
* | fa8458b - local6 (2 分钟前) <King>
* | a4f6c50 - local5 (2 分钟前) <King>
* | 6118659 - local4 (2 分钟前) <King>
* | b2bfba8 - local3 (2 分钟前) <King>
* | 0f422b5 - local2 (2 分钟前) <King>
* | 3be57eb - local1 (2 分钟前) <King>
|/  
* 491590d - commit9 (3 分钟前) <King>
* 918800c - commit8 (3 分钟前) <King>
* 0dfe536 - commit7 (3 分钟前) <King>
* 17521f8 - commit6 (3 分钟前) <King>
* 9db0b79 - commit5 (3 分钟前) <King>
* 58cbf4e - commit4 (3 分钟前) <King>
* 1c4abaa - commit3 (3 分钟前) <King>
* b973520 - commit2 (3 分钟前) <King>
* 9718254 - commit1 (3 分钟前) <King>
```

利用rebase，同样可以做的很**干净**
``` bash
king@king:/tmp/git/work$ git checkout  tmp 
切换到分支 'tmp'
king@king:/tmp/git/work$ git rebase master 
首先，回退分支以便在上面重放您的工作...
应用：local1
应用：local2
应用：local3
应用：local4
应用：local5
应用：local6
应用：local7
应用：local8
应用：local9
```



