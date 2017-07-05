---
title: Drone学习笔记
date: 2017-03-07 16:35:21
tags:
 - drone
 - golang
 - 持续集成
categories:
 - 源码学习
---

[Drone](https://github.com/drone/drone)是一个优秀的[持续集成（CL）](http://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)的系统，最大的特色就是它的执行流（pipeline）是在docker里执行，这就确保了每一次环境都保持一致，避免其他干扰。一些其他主流的包括但不限于

1. [Jenkins](https://jenkins.io/index.html)
2. [Travis](https://travis-ci.com/)
3. [Codeship](https://www.codeship.io/)
4. [Strider](https://github.com/Strider-CD/strider)

# 安装运行
可以下载编译好的二进制包，也可以下载源码编译（很简单，参考<https://github.com/drone/drone>）
``` bash
git clone git://github.com/drone/drone.git $GOPATH/src/github.com/drone/drone
cd $GOPATH/src/github.com/drone/drone

make deps          # Download required dependencies
make gen           # Generate code
make build_static  # Build the binary
```

## 运行
建议先仔细阅读官方文档<http://readme.drone.io/>，运行参考<http://readme.drone.io/admin/installation-guide/>，官方推荐的方式使用docker-compose来运行程序，包含两个后台

1. drone server，web服务
2. drone agent，代理服务，用于运行docker

其他一些组件

1. [drone/mq](https://github.com/drone/mq)，a lightweight STOMP message broker
2. drone cli，一堆的工具集合

完整的帮助如下
``` bash
king@king:~/code/go/src/github.com/drone/drone/release$ ./drone
NAME:
   drone - command line utility

USAGE:
   ./drone [global options] command [command options] [arguments...]
   
VERSION:
   0.5.0+dev
   
COMMANDS:
   agent        starts the drone agent
   agents       manage agents
   build        manage builds
   deploy       deploy code
   exec         execute a local build
   info         show information about the current user
   secret       manage secrets
   server       starts the drone server daemon
   sign         creates a secure yaml file
   repo         manage repositories
   user         manage users
   org          manage organizations
   global       manage global state
   help, h      Shows a list of commands or help for one command
   
GLOBAL OPTIONS:
   -t, --token          server auth token [$DRONE_TOKEN]
   -s, --server         server location [$DRONE_SERVER]
   --help, -h           show help
   --version, -v        print the version
```

## server
在运行服务之前，需要准备参数，可以通过命令行查看，代码中的定义见【src/github.com/drone/drone/drone/server.go】，下面是一些搭建drone hello world的参考

1. <http://www.wtoutiao.com/p/38b2N4A.html>
2. <https://overflow.no/blog/2016/11/14/self-hosted-cdci-environment-with-docker-droneio-and-drone-wall-on-debian-8/>
3. <http://tleyden.github.io/blog/2016/02/15/setting-up-a-self-hosted-drone-dot-io-ci-server/>

以github为例，必需的参数如下
1. DRONE_OPEN=true，自动创建账户
2. DRONE_GITHUB=true，使用github，可选的见【src/github.com/drone/drone/router/middleware/remote.go
3. DRONE_GITHUB_CLIENT=${DRONE_GITHUB_CLIENT},从github网站获取
4. DRONE_GITHUB_SECRET=${DRONE_GITHUB_SECRET},从github网站获取
5. DRONE_SECRET=${DRONE_SECRET}，用于agent连接server的key，具体通过drone/mq验证，使用websocket通道

``` go
// helper function to setup the remote from the CLI arguments.
func setupRemote(c *cli.Context) (remote.Remote, error) {
	switch {
	case c.Bool("github"):
		return setupGithub(c)
	case c.Bool("gitlab"):
		return setupGitlab(c)
	case c.Bool("bitbucket"):
		return setupBitbucket(c)
	case c.Bool("stash"):
		return setupStash(c)
	case c.Bool("gogs"):
		return setupGogs(c)
	default:
		return nil, fmt.Errorf("version control system not configured")
	}
}
```


不使用docker，直接写个脚本即可运行，为了使用依赖管理员权限的功能，增加DRONE_ADMIN，所有的记录存储在后端数据库，默认是sqlite3，当前目录下的drone.sqlite，为了避免在不同的目录出现问题，通过DRONE_DATABASE_DATASOURCE写死路径。

支持的数据库见【src/github.com/drone/drone/store/datastore/store.go】

``` bash
#!/bin/bash
export DRONE_OPEN=true
export DRONE_GITHUB=true
export DRONE_GITHUB_SECRET=22222
export DRONE_GITHUB_CLIENT=11111
export DRONE_SECRET=create_a_random_secret_here
export DRONE_DATABASE_DRIVER=sqlite3
export DRONE_DATABASE_DATASOURCE=/home/king/code/go/drone.sqlite
export DRONE_ADMIN=king1,king2
./drone  "${@}"

```

``` go
// helper function to setup the meddler default driver
// based on the selected driver name.
func setupMeddler(driver string) {
	switch driver {
	case "sqlite3":
		meddler.Default = meddler.SQLite
	case "mysql":
		meddler.Default = meddler.MySQL
	case "postgres":
		meddler.Default = meddler.PostgreSQL
	}
}
```

由于需要使用github的oauth认证，以及接收推送，所以服务器必须支持公网访问，若要本地测试，可以参考**[ngrok反向代理局域网开发机](http://blog.qiujinwu.com/2017/02/13/ngrok-reverse-proxy/)**

## agent

agent用于代理执行docker容器，由于docker运行需要超级管理员权限，所以需要sudo，或者切换到root。为了方便测试，最好是将当前用户添加到docker组，参考<https://my.oschina.net/zhouhui321/blog/788431>
``` bash
sudo gpasswd -a ${USER} docker
sudo service docker restart
```

agent完整参数见【src/github.com/drone/drone/drone/agent/agent.go】，必需的参数如下

1. DRONE_SERVER=ws://${drone-server}/ws/broker，server的路径
2. DRONE_SECRET=${DRONE_SECRET}，在server中配置的token

同样可以export两个环境变量之后，直接drone agent即可

## cli
运行上述两个后台之后，就可以用浏览器访问【http//${drone-server}】，若未登录点击【登录/login】会自动跳转到github进行认证，由于设置了【DRONE_OPEN=true】所以认证成功之后，会自动创建以github用户名的账户。

drone并未使用普通的session管理机制，而是使用[jwt](http://www.jianshu.com/p/576dbf44b2ae)机制，并使用cookie传输，如下

>user_sess=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjEuNDg5MDUxNjc5ZSswOSwidGV4dCI6InFqdyIsInR5cGUiOiJzZXNzIn0.SD1uwHk_E7yT-74sVXHwrLPKmonbwRtQTzT8cNPrupg
_crsf=MTQ4NzkyMTYzOHxEdi1CQkFFQ180SUFBUkFCRUFBQU12LUNBQUVHYzNSeWFXNW5EQW9BQ0dOemNtWlRZV3gwQm5OMGNtbHVad3dTQUJCS2IzSk5lbVEwWlRaV05XdHdUalZrfHeLaVDFFSGOkLA19-4Y6G5gYkzJatkOn8oNnHj4eb6q;

cli运行依赖的参数包括

1. DRONE_SERVER=http://${drone-server} ，server的地址，注意名称和agent的一样，但地址并不一致
2. DRONE_TOKEN=fsajdflkas，token就是jwt的token，注意和DRONE_SECRET区别

我们可以直接将网站的jwt token直接拿来使用，但仅限于查询操作，其他操作会因为csrf限制无权限，参考【src/github.com/drone/drone/router/middleware/session/user.go】

``` go
// if this is a session token (ie not the API token)
// this means the user is accessing with a web browser,
// so we should implement CSRF protection measures.
if t.Kind == token.SessToken {
    err = token.CheckCsrf(c.Request, func(t *token.Token) (string, error) {
        return user.Hash, nil
    })
    // if csrf token validation fails, exit immediately
    // with a not authorized error.
    if err != nil {
        c.AbortWithStatus(http.StatusUnauthorized)
        return
    }
}
```

这里实际上引申出一个session type的东西
``` go
const (
	UserToken  = "user"
	SessToken  = "sess"
	HookToken  = "hook"
	CsrfToken  = "csrf"
	AgentToken = "agent"
)
```

网页授权使用的是SessionToken，而drone cli使用UserToken，具体通过drone网站右上角【ACCOUNT】，再回到左上角【SHOW TOKEN】,参考【src/github.com/drone/drone/router/router.go】

``` go
user := e.Group("/api/user")
{
    user.POST("/token", server.PostToken)
    user.DELETE("/token", server.DeleteToken)
}
```

cli是一堆的命令集合，具体参考官方文档，有些功能仅限cli使用

## 普通账户密码登录
drone代码中有一些相关的逻辑，但并不完整，所以不可用
``` go
// src/github.com/drone/drone/router/router.go
e.GET("/login", server.ShowLogin)
e.GET("/login/form", server.ShowLoginForm)
e.GET("/logout", server.GetLogout)
```

``` go
// src/github.com/drone/drone/server/pages.go

// ShowLoginForm displays a login form for systems like Gogs that do not
// yet support oauth workflows.
func ShowLoginForm(c *gin.Context) {
	c.HTML(200, "login.html", gin.H{})
}

// src/github.com/drone/drone/server/template/files/login.html
```




