---
title: ResfulApi测试框架tavern
date: 2020-02-21 00:00:00
tags:
 - tavern
categories:
 - Python
---

# Sample

测试用例由一个测试名称、一个或多个阶段(stage)组成，每个阶段都有一个名称、一个请求和一个响应。举个简单的例子

``` yaml
test_name: Get some fake data from the JSON placeholder API

stages:
  - name: Make sure we have the right ID
    request:
      url: https://jsonplaceholder.typicode.com/posts/1
      method: GET
    response:
      status_code: 200
      body:
        id: 1
      save:
        body:
          returned_id: id
```

此用例表示, 使用`request.method`方法向`request.url`请求, 期望成功返回(`status`等于200), 并且返回的内容(json)中, id等于1, 并且把id存在变量`returned_id`中

# Request

`请求段`描述发送到服务器的内容

+ url
+ json post/put/..的json内容(如果是json格式)
+ param get方法的参数
+ data form参数
+ headers
+ method

# Response
响应段描述了我们期望得到的结果。

+ status_code 期望的status, 缺省200, 也可能201或其他
+ body 期待返回的内容
+ headers 期待返回的header
+ redirect_query_params 期望重定向URL的参数

## Save
The save block can save values from the response for use in future requests. Things can be saved from the body, headers, or redirect query parameters. When used to save something from the json body, this can also access dictionaries and lists recursively. If the response is

save块可以从响应中保存结果，以便在将来的请求中使用。可以从body、header或重定向查询参数中保存结果。当用于从json中保存内容时，它还可以递归地访问字典和列表, 获取内部某个值。如果响应是

``` json
{
    "thing": {
        "nested": [
            1, 2, 3, 4
        ]
    }
}
```

可以从响应块中的某个值保存到first val的值中
``` yaml
response:
  save:
    body:
      first_val: thing.nested[0]
```
也可以使用函数调用来保存数据,见后续

# 变量
## 全局变量
``` yaml
# common.yaml
---

name: 全局定义
description: |
  测试

variables:
  global_url: http://www.baidu.com
```
``` yaml
---
test_name: 登录/个人信息

includes:
  - !include common.yaml

stages:
  - name: 登录
    request:
      url: "{global_url:s}/user/login"

```
## 保存变量
留意`global_token`

``` yaml
stages:
  - name: 登录
    request:
      url: "{global_url:s}/user/login"
      method: POST
      json:
        username: "user"
        password: "pwd"
    response:
      status_code: 200
      save:
        body:
          global_token: data.token # 把token存下来后续调用
  - name: 个人信息
    request:
      url: "{global_url:s}/user/info"
      method: GET
      headers:
        Authorization: "{global_token}" # 获取之前存下来的token
```

## Request变量
可以从当前的request参数中获取值用作判断

留意`tavern.request_vars`

``` yaml
stages:
  - name: 登录
    request:
      url: "{global_url:s}/user/login"
      method: POST
      json:
        username: "user"
        password: "pwd"
    response:
      status_code: 200
      body:
        code: 0
        message: OK
        data:
          username: "{tavern.request_vars.json.username}" # 返回的username和参数一致
```

## 函数保存结果
``` yaml
stages:
  - name: 登录
    request:
      url: "{global_url:s}/user/login"
      method: POST
      json:
        username: "user"
        password: "pwd"
    response:
      status_code: 200
      body:
        code: 0
        message: OK
      save:
        $ext:
          function: base:save_test_data
```
``` python
# export PYTHONPATH=$PYTHONPATH:.
from box import Box

def save_test_data(response):
    return Box({"test_user_id": response.json()["data"]["userID"]})
```

## 函数校验

支持自定义参数

``` python
def check_message(response):
    assert response.json().get("message") == "OK"

def check_code(response,**kwargs):
    assert response.json().get("code") == kwargs["expect"]
```
``` yaml
stages:
  - name: 登录
    request:
      url: "{global_url:s}/user/login"
      method: POST
      json:
        username: "user"
        password: "pwd"
    response:
      status_code: 200
      verify_response_with: # 单函数判断 export PYTHONPATH=$PYTHONPATH:.
        function: base:check_message
  - name: 个人信息
    request:
      url: "{global_url:s}/user/info"
      method: GET
    response:
      status_code: 200
      body:
        code: 0
        message: OK
      verify_response_with: # 多函数判断
        - function: base:check_message
        - function: base:check_code
          extra_kwargs:
            expect: 0 # 期望 code = 0 (自定义参数)
```

## 生成header

`$ext` 表示下面的内容是函数生成器

``` python
from box import Box

def get_test_header():
    auth_header = {
        "how": "are you"
    }
    return Box(auth_header)
```
``` yaml
stages:
  - name: 登录
    request:
      url: "{global_url:s}/user/login"
      method: POST
      headers:
        $ext:
          function: base:get_test_header
```

## 环境变量

留意`tavern.env_vars`
``` yaml
stages:
  - name: 登录
    request:
      url: "{global_url:s}/user/login"
      method: POST
      headers:
        Authorization: "Basic {tavern.env_vars.SECRET_CI_COMMIT_AUTH}"
```



# 参考
1. <https://github.com/taverntesting/tavern>
2. <https://tavern.readthedocs.io/en/latest/basics.html>
3. <https://pypi.org/project/python-box/>

