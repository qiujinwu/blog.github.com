---
title: gitlab/sentry集成
date: 2020-08-01 10:36:35
tags:
 - gitlab
 - sentry
categories:
 - 开发环境
---

gitlab/sentry都通过docker在本机运行, 本机docker 接口ip地址`172.17.0.1`

参考旧文

+ [搭建gitlab测试环境](https://blog.kimq.cn/2017/06/19/gitlab-env/)
+ [Sentry](https://blog.kimq.cn/2017/03/10/sentry/)

# 运行

## 运行Sentry
为了避免端口冲突, sentry端口改成8081

```
docker run -d --name my-sentry -e SENTRY_SECRET_KEY='<KEY>' \
	-p 8081:9000 \
	--link sentry-redis:redis --link sentry-postgres:postgres sentry:9.1.2
```



## 运行Gitlab

+ 这里指定了ce的版本, 避免新版本再测试出现不一致
+ 通过参数指定了`hostname`为docker的ip

```
docker run -it --rm  \
    --hostname 172.17.0.1 \
    --publish 8080:80 --publish 2222:22 \
    --name gitlab \
    --volume /gitlab/config:/etc/gitlab \
    --volume /gitlab/logs:/var/log/gitlab \
    --volume /gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:12.9.10-ce.0
```

> 22端口容易被现有服务占用, 这里使用`2222端口`用作代码传输的ssl通道

可以修改本机的`~/.ssh/config`, 参考[这里](https://stackoverflow.com/questions/3596260/git-remote-add-with-other-ssh-port)
``` bash
$ cat ~/.ssh/config 
HOST 172.17.0.1
    Port 2222
```

# gitlab集成sentry

> 可以查看对应proj的问题,以及作出处理

就是gitlab里面使用sentry的功能, 所以需要去sentry`赋(授)能(权)`

+ 登录sentry
+ 左上角 - 个人中心 - 下拉菜单 - API key
+ 打开页面, 找到或者创建auth token, 需要`project:read, event:read` 两个权限

完整流程, 可以查看[Gitlab Error Tracking](https://docs.gitlab.com/ee/operations/error_tracking.html)

## gitlab proj启用`error tracking`
+ Navigate to your project’s `Settings > Operations`.
+ Ensure that the `Active` checkbox is set.
+ In the Sentry API URL field, enter your Sentry hostname. For example, enter https://sentry.example.com (本例`http://172.17.0.1:8081/`) if this is the address at which your Sentry instance is available. For the SaaS version of Sentry, the hostname will be https://sentry.io.
+ In the Auth Token field, enter the token you `previously generated`(见上一步生成的token).
+ Click the Connect button to test the connection to Sentry and populate the Project dropdown.
+ From the Project dropdown, choose a `Sentry project` to link to your GitLab project.
+ Click `Save changes` for the changes to take effect.
+ You can now visit `Operations > Error Tracking` in your project’s sidebar to view a list of Sentry errors.

到此, 我们可以在gitlab看到对应项目的所有senry 问题记录, 对于`gitlab-ce:12.9.10-ce.0` , 最上面可以`查看/过滤`问题, 也可以`忽略/解决`问题, 也可以基于问题新建一个`gitlab issue`

> 这些操作需要额外的`event:admin` 权限

而更早的`gitlab-ce:12.3.5-ce.0` 就只有一个简单的列表

> 对于本机自建的gitlab, 有可能报错`Connection has failed. Re-check Auth Token and try again`, 参考[这里](https://gitlab.com/gitlab-org/gitlab/-/issues/32314)

GitLab -> Admim area -> Settings -> Network -> Outbound requests -> Allow requests to the local network from hooks and services

By checking both boxes, everything worked as documented.

# Sentry 集成 gitlab

> 可以直接在sentry创建gitlab issue

+ In Sentry, navigate to Organization `Settings > Integrations`.
+ Within Integrations, find the `GitLab icon` and click Install.
+ In the resulting modal, click `Add Installation`.
+ In the pop-up window, complete the instructions to create a Sentry app within GitLab. Once you’re finished, click Next.
+ Fill out the resulting GitLab Configuration form with the following information:
  - The GitLab URL is the base URL for your GitLab instance. If using gitlab.com, enter <http://172.17.0.1:8080>
  - Find the GitLab Group Path in your group’s GitLab page. 自定, 必须gitlab存在
  - Find the GitLab Application ID and Secret in the Sentry app within GitLab.
    * GitLab Application ID : gitlab生成 - GitLab Application Secret : gitlab生成
  - Use this information to fill out the GitLab Configuration and click Submit.
+ In the resulting panel, click `Authorize`.
+ In Sentry, return to Organization Settings > Integrations. You’ll see a new instance of GitLab underneath the list of integrations.
+ Next to your GitLab Instance, click Configure. It’s important to configure to receive the full benefits of commit tracking.
+ On the resulting page, click Add Repository to select which repositories in which you’d like to begin tracking commits.

完整记录参考[integrations gitlab](https://docs.sentry.io/workflow/integrations/gitlab/)

## 生成gitlab application

具体的流程参考 [GitLab as OAuth2 authentication service provider](https://docs.gitlab.com/ee/integration/oauth_provider.html)

+ Name : Sentry (自定义)
+ Redirect URI : <http://172.17.0.1:8081/extensions/gitlab/setup/>
+ Scopes : `api`



# 参考
+ <https://docs.gitlab.com/ee/operations/error_tracking.html>
+ <https://docs.gitlab.com/ee/integration/oauth_provider.html>
+ <https://docs.sentry.io/workflow/integrations/gitlab/>
+ <https://gitlab.com/gitlab-org/gitlab/-/issues/32314>
