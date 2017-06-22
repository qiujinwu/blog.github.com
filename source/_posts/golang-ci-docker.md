---
title: Golang持续集成，并生成docker镜像
date: 2017-6-22 18:23:16
tags:
 - golang
 - ci
categories:
 - Golang
---

本文代码请参考<https://github.com/qjw/git-notify>，部分脚本参考**[drone](https://github.com/drone/drone)**的实现

将讲述两种ci，分别是**[drone](https://github.com/drone/drone)**和**[travis-ci](https://www.travis-ci.org)**

# 编译
``` Makefile
.PHONY: build

ifneq ($(shell uname), Darwin)
	EXTLDFLAGS = -extldflags "-static" $(null)
else
	EXTLDFLAGS =
endif

PACKAGES = $(shell go list ./... | grep -v /vendor/)

deps: deps_backend

deps_backend:
	go get -u github.com/kardianos/govendor
	go get -u github.com/jteeuwen/go-bindata/...
	govendor sync

gen: gen_template

gen_template:
	go generate github.com/qjw/git-notify/templates/

test:
	go test -cover $(PACKAGES)

build: build_static

build_static:
	CGO_ENABLED=0 GOOS=linux  go install -a -installsuffix cgo \
    	-ldflags '${EXTLDFLAGS}' github.com/qjw/git-notify
	mkdir -p release
	cp $(GOPATH)/bin/git-notify release/
```

1. makefile中的项目路径需要根据实际情况修改
2. 依赖的二进制工具（**deps_backend**）也因项目而异
3. 上面的**govendor sync**是用来根据**vendor/vendor.json**来同步依赖库，若使用其他，需要酌情修改
4. **[git-notify](https://github.com/qjw/git-notify)**有模板资源，并且依赖<github.com/jteeuwen/go-bindata>打包到代码中，所以需要go template，根据需要作删减

有了这个，运行下面命令即可在项目主目录/release生成目标执行文件。**为了生成的docker镜像更小，这里使用静态编译**，若无此需求，可以普通编译，以减少目标文件尺寸
``` bash
make deps gen test build
```

# Dockerfile
编译成功之后，就可以运行下列命令生成docker镜像
``` bash
docker build -t git-notify:0.1 .
```

``` Dockerfile
FROM scratch
ADD release/git-notify /
CMD ["/git-notify"]
```

# [drone](https://github.com/drone/drone)
``` yaml
workspace:
  base: /go
  path: src/github.com/qjw/git-notify

pipeline:
  test:
    image: golang:1.8
    environment:
      - GO15VENDOREXPERIMENT=1
    commands:
      - make deps gen
      - make test

  compile:
    image: golang:1.8
    environment:
      - GO15VENDOREXPERIMENT=1
      - GOPATH=/go
    commands:
      - export PATH=$PATH:$GOPATH/bin
      - make build

  docker:
    image: plugins/docker
    username: ${EMAIL}
    password: ${PASSWORD}
    email: ${EMAIL}
    repo: hub.c.163.com/${USERNAME}/test
    registry: hub.c.163.com
    tags: latest
```

# [travis-ci](https://www.travis-ci.org)
``` yaml
sudo: required
language: go
go:
 - 1.8

services:
  - docker

branches:
  only:
    - master

install:
  - make deps gen
  - make test

script:
  - make build

after_script:
  - docker build -t qjw/git-notify .
  - docker login -u ${EMAIL} -p ${PASSWORD} hub.c.163.com
  - docker tag qjw/git-notify hub.c.163.com/${USERNAME}/git-notify
  - docker push hub.c.163.com/${USERNAME}/git-notify
```