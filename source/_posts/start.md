---
title: 新博客之旅
date: 2017-02-13 09:32:16
tags:
 - hexo
 - travis
categories:
 - 运营运维
 - 网站
---

## 选型
之前有个博客用[jekyll](https://github.com/jekyll/jekyll)搞的，但是它依赖ruby，这在Windows下很麻烦，而且ruby个人感觉很另类不喜欢（个人喜好问题），而且编译又慢，所以果断用[hexo](https://github.com/hexojs/hexo)


## 安装
### 安装git，node/npm

创建编写博客的目录，使用npm安装必须的组件

``` bash
npm install hexo-cli -g
npm install hexo --save
hexo -v
```

``` bash
king@king:~/tmp/blog$ hexo -v
hexo: 3.2.2
hexo-cli: 1.0.2
os: Linux 4.4.0-47-generic linux x64
http_parser: 2.5.0
node: 4.2.6
v8: 4.5.103.35
uv: 1.8.0
zlib: 1.2.8
ares: 1.10.1-DEV
icu: 55.1
modules: 46
openssl: 1.0.2g-fips
```

### 初始化博客

``` bash
hexo init
# 安装依赖
npm install
# 编译
hexo g
# 运行
hexo s
```

``` bash
king@king:~/tmp/blog$ hexo s
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```

### 配置
可以在 _config.yml 中修改大部份的配置。

参数 | 描述
--- | ---
`title` | 网站标题
`subtitle` | 网站副标题
`description` | 网站描述
`author` | 您的名字

参数 | 描述
--- | ---
`theme` | 当前主题名称。值为`false`时禁用主题
`deploy` | 部署部分的设置

### 主题
见[A simple Bootstrap v3 blog theme for Hexo](https://github.com/cgmartin/hexo-theme-bootstrap-blog)
> https://github.com/cgmartin/hexo-theme-bootstrap-blog

``` yaml
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: bootstrap-blog
```

### 新文章

``` bash
king@king:~/tmp/blog$ hexo new "start"
INFO  Created: ~/tmp/blog/source/_posts/start.md
```

### tag和category
tag支持多个，category支持层级关系

``` yaml
tags:
 - hexo
 - travis
categories:
 - 运营运维
 - 网站
```

## 域名
新建文件**source/CNAME**

``` bash
king@king:~/blog$ cat source/CNAME 
blog.qiujinwu.com
```

然后在域名管理网站，例如[Dnspod](https://www.dnspod.cn)，新建CNAME记录。【*xxx就是github用户名,注意最后面的点*】

主机记录 | 记录类型 | 记录值
--- | --- | ---
`blog` | CNAME | xxx.github.com.


### 发布
hexo g编译之后，默认会在public目录生成编译好的前端内容，发布可以简单地将这个目录push到github目标仓库的master分支。

``` yaml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:yyy/xxx.git
  branch: master
```

运行下面命令发布
``` bash
hexo d
```

## 发布
前面的发布有个问题，原始文件没有备份。最简单的办法是写个脚本，把源文件和编译好的文件分别push到不同的仓库。若不想别人看到源文件，还可以将源文件的残酷设置为私有（需要掏银子）

另一种办法是将源文件和编译好的文件放在不同的分支，参考[Hexo利用Github分支在不同电脑上写博客](http://www.dxjia.cn/2016/01/27/hexo-write-everywhere/)

另一种办法是用持续集成，推荐[travis-ci](https://travis-ci.org/),参考[手把手教你使用Travis CI自动部署你的Hexo博客到Github上](http://www.jianshu.com/p/e22c13d85659)

注册github账户，新建两个仓库，一个用于存储博客源文件，一个用于存储编译后的静态内容。对于后者，开启**gh-pages**功能【*注意：只有master分支支持gh-pages功能*】

用github账户登录travis-ci之后，按照前面[参考链接](http://www.jianshu.com/p/e22c13d85659)配置好即可，本质上就是push到github的源文件仓库，travis-ci就会收到通知，然后pull下来，并且按照.travis.yml配置的参数执行。

在我的环境下，.travis.yml文件中的命令无法读取到<https://travis-ci.org/>网站配置的环境变量，所以用另一种加密token的办法实现的，具体参考[Hexo 自动部署到 Github](http://lotabout.me/2016/Hexo-Auto-Deploy-to-Github/)

``` bash
# 安装ruby之后
gem install travis
# cd 博客项目文件夹根目录
travis login --auto
# 加密从github取下来的token，不需要<>
# 此操作会自动往.travis.yml添加一个密钥[env - global - secure]
travis encrypt 'GH_TOKEN=<TOKEN>' --add
```

完整的文件如下
``` yaml
language: node_js
node_js: stable
install:
- npm install
script:
- hexo g
after_script:
- cd ./public
- git init
- git config user.name "king"
- git config user.email "qiujinwu456@gmail.com"
- git add .
- git commit -m "Update docs"
- git push --force --quiet "https://$GH_TOKEN@${GH_REF}" master:master
branches:
  only:
  - master
env:
  global:
  - GH_REF: github.com/xxxx/blog.qiujinwu.com.git
  - secure: hmLkN8N2DZ9x/auC9hpdnv/he1XBYtxG/T3z
```

## RSS
参考[Hexo—正确添加RSS订阅](http://hanhailong.com/2015/10/08/Hexo%E2%80%94%E6%AD%A3%E7%A1%AE%E6%B7%BB%E5%8A%A0RSS%E8%AE%A2%E9%98%85/)

``` bash
# 安装hexo-generator-feed
npm install hexo-generator-feed --save
```

配置到根目录的_config.yml
``` yaml
plugin:
- hexo-generator-feed
#Feed Atom
feed:
  type: atom
  path: atom.xml
  limit: 20
```

最后在主题的配置中添加入口

## 评论
使用[disqus](https://disqus.com)，

进入disqus网站，填写博客的域名，按照指引完成之后，获取shortname，然后在_config.xml中添加

``` yaml
disqus_shortname: shortname
```

## 参考
1. <https://xuanwo.org/2015/03/26/hexo-intor/>
1. <http://jiji262.github.io/2016/04/15/2016-04-15-hexo-github-pages-blog/>
1. <https://github.com/hexojs/hexo>
1. <https://github.com/cgmartin/hexo-theme-bootstrap-blog>
2. <http://lotabout.me/2016/Hexo-Auto-Deploy-to-Github>
