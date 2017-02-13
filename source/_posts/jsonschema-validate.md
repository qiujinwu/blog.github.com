---
title: 使用jsonschema作参数校验
date: 2017-02-13 19:44:52
tags:
 - jsonschema
categories:
 - 后端
---

jsonschema是一个校验库，比如[python 版本](https://pypi.python.org/pypi/jsonschema)，也有[go 版本](https://github.com/xeipuuv/gojsonschema)。目前前后端分离框架，一般会使用json作为传参的格式，所以用它来作校验也未尝不是一个办法，因为

1. 规则是标准化的东西，不用去学习一套套的另类规则
2. 规则可以写到json文件甚至用yaml格式，而不用硬编码到代码中(*其他校验方案也有不需要硬编码的*)
3. 在启动时直接cache到内存以提高效率，也可以动态的更新规则
4. 可以很好地和[swagger ui](https://github.com/swagger-api/swagger-ui)配合使用

为了支持上述论点，基于[flasgger](https://github.com/rochacbruno/flasgger)开发了项目<https://github.com/qjw/flasgger>。在前者的基础上做了如下修改

1. 原版本的flasgger使用yaml来编写文档，本项目改成json格式（*因为swagger-ui不认yaml*）
2. 完善ref，支持对象定义的相对引用，根目录在配置中指定
3. 加强校验支持，和swagger使用同一份json定义。增加swag_from注解，提供文档/校验的全局开关和接口独立开关
4. 支持自定义规则，已规避正则费解和管理的麻烦
5. 支持自动过滤空置，提高前端开发效率
6. 自定义错误提示

完整的说明见[README](https://github.com/qjw/flasgger)


