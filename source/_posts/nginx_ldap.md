---
title: nginx配置ldap登录授权
date: 2020-02-04 00:00:00
tags:
 - openldap
 - nginx
categories:
 - Openldap
---

本文代码来自<https://github.com/nginxinc/nginx-ldap-auth>

nginx内建了`auth_request`实现了权限控制拦截功能, 这适用于需要在无认证的服务前增加访问控制, 比如`Kibana`无默认的权限控制, 但是公司的日志显然不能裸奔

实现包含三部分
+ `nginx`(`auth_request`)
+ (较通用)的`认证服务`
+ 实际的`业务服务`

# 业务服务
文中的业务服务比较简单, 用Python写了个简单的web服务, 以及用于客户输入`帐号密码`的表单页面

# Nginx
+ nginx 绑定端口8081 供外部访问
+ `auth_request`做全量拦截, 如果认证失败, 抛出`401`
+ nginx 将401重定向到/login, 即`业务服务`中的表单
+ 表单提交又回到`auth_request`
  - 若存在cookie, 从cookie中解码出帐号密码, 然后ldap认证
  - 对于首次登录, 则从form表单中获取帐号密码
  - 认证成功,将帐号密码保存在cookie中, 避免重复输入帐号密码

``` nginx
upstream backend {
    server 127.0.0.1:9000;
}
server {
    listen 8081;

    location / {
        auth_request /auth-proxy;
        error_page 401 =200 /login;
        proxy_pass http://backend/;
    }

    location /login {
        proxy_pass http://backend/login;
        proxy_set_header X-Target $request_uri;
    }

    location = /auth-proxy {
        internal;

        proxy_pass http://127.0.0.1:8888;

        proxy_pass_request_body off;
        proxy_set_header Content-Length "";

        proxy_cache_key "$http_authorization$cookie_nginxauth";

        proxy_set_header X-Ldap-URL      "ldap://127.0.0.1";

        proxy_set_header X-Ldap-BaseDN   "ou=lixin,dc=example,dc=org";

        proxy_set_header X-Ldap-BindDN   "cn=admin,ou=tech,ou=lixin,dc=example,dc=org";

        proxy_set_header X-Ldap-Template "(cn=%(username)s)";

        proxy_set_header X-Ldap-BindPass "123456";

        proxy_set_header X-CookieName "nginxauth";
        proxy_set_header Cookie nginxauth=$cookie_nginxauth;
    }
}
```

> `nginx` 将所有ldap的参数,以`header`的形式传给`认证服务`, 这样确保`认证服务`相对通用


# 认证服务
+ 设定一个header头-参数的映射表, 以及服务启动传入的缺省值
+ 每次请求,都动态获取这些参数, 连同(从form表单/cookie获取到的)帐号密码存入临时变量ctx
+ 使用ctx的参数做ldap认证
+ 参数错误/认证失败 返回401

# 问题
这显然只是一个sample, 不可能在生产环境中实施, 因为安全问题和性能问题比较明显

首先将帐号密码存在cookie中显然不合适, 可以考虑对帐号密码加密再写cookie

更好的方法是将帐号密码存在内部存储, 比如redis中,然后把redis的key写cookie

此时仍然存在问题, 因为每个请求都要走一遍ldap认证, 对响应时延影响较大

一种折中的方案是, 认证成功之后, 生成一个session并存入redis, session_id写cookie, 用户完整信息在session中, 但仍有不足, 比如ldap删除某个用户, 并不会立即反映到业务系统来
