---
title: Python安装Git库
date: 2018-07-16 17:33:35
tags:
 - python
 - pip
 - git
categories:
 - 后端
---

依赖特定的版本

例如requirements.txt中
``` bash
# ssh方式
git+ssh://git@github.com/qjw/flask-swagger.git@v1.0.1
# https方式
git+https://github.com/qjw/flask-swagger.git@@v1.0.1
# 本地文件系统
git+file:///home/king/code/flask-swagger/
```
或者
``` bash
pip install git+https://github.com/qjw/flask-swagger.git@v1.0.1
```

安装之后,源码在目录`venv/lib/python3.5/site-packages/flask_swagger`

@后面可以是`tag`/`branch`甚至`commit`.

这种方案的不足是: 每次提交修改都得`打tag`,若是提交频繁会相当麻烦且容易出错,

如果改动之后,不重新打tag,会受到cache的影响而不能即使更新

``` bash
mkdir -p pip-cache
pip install --cache-dir pip-cache -r requirements.txt

```


## 调试
在开发期间,可以使用`文件系统的方式`不经过外网安装

## 频繁的改动
前面打tag的方式,适合那些不经常变化的基础库,对于频繁有更新的项目,可以考虑使用[`editable` packages](https://pip.readthedocs.io/en/1.1/requirements.html)方式安装

``` bash
# 指定版本(tag)
-e git+ssh://git@github.com/qjw/flask-swagger.git@v1.0.1#flask-swagger
# 直接从master
-e git+ssh://git@github.com/qjw/flask-swagger.git#egg=flask-swagger
# 从本地目录引用(调试)
-e git+file:///home/king/code/flask-swagger#egg=flask-swagger
```

和前面的方式相比,多了参数`-e`,以及后面额`egg` hash,后者指定了源码存放的目录,具体的路径在`venv/src/flask-swagger`.

### 已知问题
若直接使用master或其他分支,会产生历史版本不一致的问题

> 比如我重新编译一个历史版本,会因为依赖变化导致不一致的行为

每一次产生一个branche是不现实的,用tag相对较好.代码管理是一个受权限控制的行为,那么这个tag由谁来打就是个问题

1. 如果开发者自行打tag,那git tags会各种乱和冲突
2. 如果管理员来做,那么开发效率有严重影响(在联调过程中,tag必须准备好,可是改个bug又必须重新tag)

直接使用commit id也是一种可行的办法,不过这种方式非常的不优雅