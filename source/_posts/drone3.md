---
title: Drone学习笔记3
date: 2017-3-8 11:04:27
tags:
 - drone
 - golang
 - 持续集成
categories:
 - 源码学习
---

# 入口
drone入口在【src/github.com/drone/drone/drone/main.go】，整个目录【src/github.com/drone/drone/drone】都是各种入口，包括drone server/agent和各种cli调用，除了server/agent，cli都是对【src/github.com/drone/drone/client】的一个封装调用。

每一个命令行调用都包含一堆的参数，并且子命令又包含一堆参数，这依赖于**[urfave/cli](https://github.com/urfave/cli)**。注意main函数的app变量。

``` go
func main() {
	envflag.Parse()

	app := cli.NewApp()
	app.Name = "drone"
	app.Version = version.Version
	app.Usage = "command line utility"
	app.Flags = []cli.Flag{
		cli.StringFlag{
			Name:   "t, token",
			Usage:  "server auth token",
			EnvVar: "DRONE_TOKEN",
		},
		cli.StringFlag{
			Name:   "s, server",
			Usage:  "server location",
			EnvVar: "DRONE_SERVER",
		},
	}
	app.Commands = []cli.Command{
		agent.AgentCmd,
		agentsCmd,
		buildCmd,
		deployCmd,
		execCmd,
		infoCmd,  //-------------------
		secretCmd,
		serverCmd,
		signCmd,
		repoCmd,
		userCmd,
		orgCmd,
		globalCmd,
	}

	app.Run(os.Args)
}
```

src/github.com/drone/drone/drone/info.go
``` go
var infoCmd = cli.Command{
	Name:  "info",
	Usage: "show information about the current user",
	Action: func(c *cli.Context) {
		if err := info(c); err != nil {
			log.Fatalln(err)
		}
	},
	Flags: []cli.Flag{
		cli.StringFlag{
			Name:  "format",
			Usage: "format output",
			Value: tmplUserInfo,
		},
	},
}
```

# Cli
drone cli仅仅是对【src/github.com/drone/drone/client】的一个封装调用，本质上就是一个http请求，只不过使用不同的jwt token。

src/github.com/drone/drone/drone/user_list.go
``` go
func userList(c *cli.Context) error {
	client, err := newClient(c)
	if err != nil {
		return err
	}

	users, err := client.UserList() //---------------------
	if err != nil || len(users) == 0 {
		return err
	}
```

src/github.com/drone/drone/client/client.go
``` go
// Client is used to communicate with a Drone server.
type Client interface {
	// Self returns the currently authenticated user.
	Self() (*model.User, error)

	// User returns a user by login.
	User(string) (*model.User, error)

	// UserList returns a list of all registered users.
	UserList() ([]*model.User, error)
```

src/github.com/drone/drone/client/client_impl.go
``` go

const (
	pathUsers         = "%s/api/users"
)

// UserList returns a list of all registered users.
func (c *client) UserList() ([]*model.User, error) {
	var out []*model.User
	uri := fmt.Sprintf(pathUsers, c.base)

	log.Print("user list uri %s\n",uri)
	err := c.get(uri, &out)
	return out, err
}
```

# 路由
drone的路由实现在【src/github.com/drone/drone/router/router.go】，使用**[gin-gonic/gin](https://github.com/gin-gonic/gin)**。

src/github.com/drone/drone/drone/server.go
``` go
func server(c *cli.Context) error {

	// setup the server and start the listener
	handler := router.Load(
		ginrus.Ginrus(logrus.StandardLogger(), time.RFC3339, true),
		middleware.Version,
		middleware.Config(c),
		middleware.Cache(c),
		middleware.Store(c),
		middleware.Remote(c),
		middleware.Agents(c),
		middleware.Broker(c),
	)
```

src/github.com/drone/drone/router/router.go
``` go

// Load loads the router
func Load(middleware ...gin.HandlerFunc) http.Handler {
	e.Use(header.NoCache)
	e.Use(header.Options)
	e.Use(header.Secure)
	e.Use(middleware...)
	e.Use(session.SetUser())
	e.Use(token.Refresh)

	e.GET("/login", server.ShowLogin)
	e.GET("/login/form", server.ShowLoginForm)
	e.GET("/logout", server.GetLogout)
	e.NoRoute(server.ShowIndex)

	// TODO above will Go away with React UI

	user := e.Group("/api/user")
	{
		user.Use(session.MustUser())
		user.GET("", server.GetSelf)
		user.GET("/feed", server.GetFeed)
		user.GET("/repos", server.GetRepos)
		user.GET("/repos/remote", server.GetRemoteRepos)
		user.POST("/token", server.PostToken)
		user.DELETE("/token", server.DeleteToken)
	}
```

这其中有一个有意思的用法，用以支持**同一个group内部分接口适用的middleware，但又不需要每一个适用的接口写一次**。
``` go
	repos := e.Group("/api/repos/:owner/:name")
	{
		repos.POST("", server.PostRepo)

		repo := repos.Group("")
		{
			repo.Use(session.SetRepo())
			repo.Use(session.SetPerm())
			repo.Use(session.MustPull)

			repo.GET("", server.GetRepo)
		}
	}
```

这其中最关键的就是各种midlleware的使用，其中**大量使用闭包用于支持全局的初始化**

# 授权
通过下面的中间件设置当前的用户，如果没有无所谓，具体的校验在后面
``` go
func SetUser() gin.HandlerFunc {
	return func(c *gin.Context) {
		var user *model.User
		// 获取token，并验证
		t, err := token.ParseRequest(c.Request, func(t *token.Token) (string, error) {
			var err error
			// 继续查询数据库是否存在用户
			user, err = store.GetUserLogin(c, t.Text)
			return user.Hash, err
		})
		if err == nil {
			confv := c.MustGet("config")
			if conf, ok := confv.(*model.Config); ok {
				// 是否admin
				user.Admin = conf.IsAdmin(user)
			}

			// 最后设置当前的user到gin.Context
			c.Set("user", user)

			// 普通的web token，需要验证csrf token
			if t.Kind == token.SessToken {
				err = token.CheckCsrf(c.Request, func(t *token.Token) (string, error) {
					return user.Hash, nil
				})
			}
		}
		c.Next()
	}
}
```

获取当前user就比较简单
``` go
func User(c *gin.Context) *model.User {
	// 从gin.Context获取当前的user，可能不存在
	// 参考对比 SetUser c.Set("user", user)
	v, ok := c.Get("user")
	if !ok {
		return nil
	}
	u, ok := v.(*model.User)
	if !ok {
		return nil
	}
	return u
}
```

通过下面两个中间件用于授权认证
``` go
func MustRepoAdmin() gin.HandlerFunc {
	return func(c *gin.Context) {
		user := User(c)
		perm := Perm(c)
		switch {
		case user == nil:
			c.String(401, "User not authorized")
			c.Abort()
		case perm.Admin == false:
			c.String(403, "User not authorized")
			c.Abort()
		default:
			c.Next()
		}
	}
}

func MustUser() gin.HandlerFunc {
	return func(c *gin.Context) {
		user := User(c)
		switch {
		case user == nil:
			c.String(401, "User not authorized")
			c.Abort()
		default:
			c.Next()
		}
	}
}
```

# 全局配置
前面提到一个DRONE_OPEN的环境变量，drone使用一个全局配置对象保存这些配置，但是和普通的全局对象不同的时，它利用了gin.Context的Context。也就是上面保存当前用户的方式。

总之，可变（比如当前登录用户）和不变（全局配置）都使用同样的方式，即通过一个闭包获取/保存对象到gin.Context，然后在用的时候从gin.Context获取。

src/github.com/drone/drone/router/middleware/config.go
``` go
func Config(cli *cli.Context) gin.HandlerFunc {
	v := setupConfig(cli)
	return func(c *gin.Context) {
		c.Set(configKey, v)
	}
}

// helper function to create the configuration from the CLI context.
func setupConfig(c *cli.Context) *model.Config {
	return &model.Config{
		Open:   c.Bool("open"),
		Yaml:   c.String("yaml"),
		Shasum: c.String("yaml") + ".sig",
		Secret: c.String("agent-secret"),
		Admins: sliceToMap(c.StringSlice("admin")),
		Orgs:   sliceToMap(c.StringSlice("orgs")),
	}
}
```

当使用oauth登录之后，回调如下
``` go
auth := e.Group("/authorize")
{
    auth.GET("", server.GetLogin)
    auth.POST("", server.GetLogin)
    auth.POST("/token", server.GetLoginToken)
}
```

src/github.com/drone/drone/server/login.go
``` go
func GetLogin(c *gin.Context) {
    // 获取全局配置对象
	config := ToConfig(c)

	// get the user from the database
	u, err := store.GetUserLogin(c, tmpuser.Login)
	if err != nil {

		// if self-registration is disabled we should return a not authorized error
        // 检查
		if !config.Open && !config.IsAdmin(tmpuser) {
			logrus.Errorf("cannot register %s. registration closed", tmpuser.Login)
			c.Redirect(303, "/login?error=access_denied")
			return
		}
	}
}
```

# 数据库
src/github.com/drone/drone/router/middleware/store.go
``` go
// Store is a middleware function that initializes the Datastore and attaches to
// the context of every http.Request.
func Store(cli *cli.Context) gin.HandlerFunc {
	v := setupStore(cli)
	return func(c *gin.Context) {
    	// 将数据库对象设置到gin.Context
		store.ToContext(c, v)
		c.Next()
	}
}

// helper function to create the datastore from the CLI context.
func setupStore(c *cli.Context) store.Store {
	return datastore.New(
		c.String("driver"),
		c.String("datasource"),
	)
}
```

src/github.com/drone/drone/store/datastore/store.go
``` go
// New creates a database connection for the given driver and datasource
// and returns a new Store.
func New(driver, config string) store.Store {
	return From(
		open(driver, config),
	)
}

// helper function to setup the meddler default driver
// based on the selected driver name.
func setupMeddler(driver string) {
	// 根据配置，获取实际的数据库对象
	switch driver {
	case "sqlite3":
		meddler.Default = meddler.SQLite
	case "mysql":
		meddler.Default = meddler.MySQL
	case "postgres":
		meddler.Default = meddler.PostgreSQL
	}
}

// helper function to setup the databsae by performing
// automated database migration steps.
func setupDatabase(driver string, db *sql.DB) error {
	var migrations = &migrate.AssetMigrationSource{
		Asset:    ddl.Asset,
		AssetDir: ddl.AssetDir,
		Dir:      driver,
	}
    // UP 确保最新，如果没有就创建表什么的
	_, err := migrate.Exec(db, driver, migrations, migrate.Up)
	return err
}

// open opens a new database connection with the specified
// driver and connection string and returns a store.
func open(driver, config string) *sql.DB {
	db, err := sql.Open(driver, config)
	if err != nil {
		logrus.Errorln(err)
		logrus.Fatalln("database connection failed")
	}
	if driver == "mysql" {
		// per issue https://github.com/go-sql-driver/mysql/issues/257
		db.SetMaxIdleConns(0)
	}

    // 设置数据库
	setupMeddler(driver)

    // 确保数据库/表是最新的
	if err := setupDatabase(driver, db); err != nil {
		logrus.Errorln(err)
		logrus.Fatalln("migration failed")
	}
}
```

## Migrate
使用**[rubenv/sql-migrate](https://github.com/rubenv/sql-migrate)**，一些类似的主流框架包括**[pressly/goose](https://github.com/pressly/goose)**

很多migrate框架都可以用自己的格式写migrate脚本，[flask-migrate](https://flask-migrate.readthedocs.io/en/latest/)甚至可以自己根据model来生成migrate脚本。这样的好处是不用sql那么啰嗦，但问题也不少

1. 需要学习它的特定语法
2. 大部分都不完善，容易被坑
3. flask-migrate生成的脚本有时候并不完善，需要在手动改改

所以用sql写migrate挺好，

## orm
drone没有使用大而全的orm框架，而是一个小巧的**[russross/meddler](https://github.com/russross/meddler)**

src/github.com/drone/drone/store/datastore/users.go
``` go
func (db *datastore) GetUserLogin(login string) (*model.User, error) {
	var usr = new(model.User)
	var err = meddler.QueryRow(db, usr, rebind(userLoginQuery), login)
	return usr, err
}

const userLoginQuery = `
SELECT *
FROM users
WHERE user_login=?
LIMIT 1
`
```

一些相对完善的如**[jinzhu/gorm](https://github.com/jinzhu/gorm)**，文档也不错<http://jinzhu.me/gorm/>，但还是要学习那一套语法，要命的是这种自定义的语法存在潜在的风险，可能生成的不对的sql或者低效的sql脚本。相对完善的orm如**[sqlalchemy](https://www.sqlalchemy.org/)**，但要学习那一套复杂的语法也并不容易。

drone由于查询较为简单，所有的查询都封装在【src/github.com/drone/drone/store】，对外只暴露接口Store

src/github.com/drone/drone/store/store.go
``` go
type Store interface {
	// GetUser gets a user by unique ID.
	GetUser(int64) (*model.User, error)

	// GetUserLogin gets a user by unique Login name.
	GetUserLogin(string) (*model.User, error)

	// GetUserList gets a list of all users in the system.
	GetUserList() ([]*model.User, error)
```

# git后端适配
由于drone支持多种git后端，例如github，gitlib等，所以对此做了一层封装，代码在【src/github.com/drone/drone/remote】，和数据库一样，只暴露了接口

src/github.com/drone/drone/remote/remote.go
``` go
type Remote interface {
	// Login authenticates the session and returns the
	// remote user details.
	Login(w http.ResponseWriter, r *http.Request) (*model.User, error)

	// Auth authenticates the session and returns the remote user
	// login for the given token and secret
	Auth(token, secret string) (string, error)
```

具体的初始化和设置在【src/github.com/drone/drone/router/middleware/remote.go】
``` go
// Remote is a middleware function that initializes the Remote and attaches to
// the context of every http.Request.
func Remote(c *cli.Context) gin.HandlerFunc {
	v, err := setupRemote(c)
	if err != nil {
		logrus.Fatalln(err)
	}
	return func(c *gin.Context) {
		remote.ToContext(c, v)
	}
}

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

# websocket
drone有两个地方用到了websocket

1. 网站动态刷新内容时，例如编译打包的日志输出
2. agent和server通信（drone/mq运行在websocket之上）

入口在【src/github.com/drone/drone/router/router.go】
``` go
ws := e.Group("/ws")
{
    ws.GET("/broker", server.Broker)
    ws.GET("/feed", server.EventStream)
    ws.GET("/logs/:owner/:name/:build/:number",
        session.SetRepo(),
        session.SetPerm(),
        session.MustPull,
        server.LogStream,
    )
}
```

一个简单的例子

src/github.com/drone/drone/server/stream.go
``` go
// LogStream streams the build log output to the client.
func LogStream(c *gin.Context) {
	// 创建server
	ws, err := upgrader.Upgrade(c.Writer, c.Request, nil)
	if err != nil {
		if _, ok := err.(websocket.HandshakeError); !ok {
			logrus.Errorf("Cannot upgrade websocket. %s", err)
		}
		return
	}

    // 创建沟通的channel
	logs := make(chan []byte)
	done := make(chan bool)
	var eof bool
	dest := fmt.Sprintf("/topic/logs.%d", job.ID)
	client, _ := stomp.FromContext(c)

    // 订阅特定的topic，用来接收agent通过drone/mq发送过来的日志
	sub, err := client.Subscribe(dest, stomp.HandlerFunc(func(m *stomp.Message) {
		if m.Header.GetBool("eof") {
			eof = true
			done <- true
		} else if eof {
			return
		} else {
            // 在mq的回调中 通过golang的channel传递出来
			logs <- m.Body
		}
		m.Release()
	}))

	for {
		select {
		case buf := <-logs:
            // 收到消息后，通过ws发送到浏览器客户端
			ws.SetWriteDeadline(time.Now().Add(writeWait))
			ws.WriteMessage(websocket.TextMessage, buf)
		case <-done:
			return
		case <-ticker.C:
			err := ws.WriteControl(websocket.PingMessage, []byte{}, time.Now().Add(writeWait))
			if err != nil {
				return
			}
		}
	}
}
```

不得不说，golang的goruntime和channel解决了C后台程序员写并发程序的痛点。

websocket服务器使用了**[gorilla/websocket](https://github.com/gorilla/websocket)**，这个**[gorilla](https://github.com/gorilla)**提供了不少非常优秀的组件，是golang写web程序的首选。

**[gorilla/websocket](https://github.com/gorilla/websocket)**的用法可以参考<https://github.com/gorilla/websocket/tree/master/examples/chat>，当然它也提供了websocket的客户端。


# mq
drone server和agent使用一个**[mq](https://github.com/drone/mq)**来通信，底层使用websocket而不是tcp。这个mq使用STOMP协议，是一个典型的订阅发布模型，并且提供确认机制。

和**[rabbitmq](https://www.rabbitmq.com/)**之类的mq不同在于用使用go语言编写，所以可以集成到服务内部，而不用单独开启一个服务，这一点和**[zeromq](http://zeromq.org/)**有点类似。关于和其他主流的mq对比，见官网<http://mq.drone.io/overview/>

## server mq初始化
通过使用闭包完成全局初始化，然后通过中间件设置到gin.Context

src/github.com/drone/drone/router/middleware/broker.go
``` go
// Broker is a middleware function that initializes the broker
// and adds the broker client to the request context.
func Broker(cli *cli.Context) gin.HandlerFunc {
    // 参见环境变量DRONE_SECRET
	secret := cli.String("agent-secret")
	if secret == "" {
		logrus.Fatalf("fatal error. please provide the DRONE_SECRET")
	}

    // 启动mq server
	broker := server.NewServer(
		server.WithCredentials("x-token", secret),
	)
    // 创建mq client
	client := broker.Client()

	var once sync.Once
	return func(c *gin.Context) {
        // 设置到gin.Context
		c.Set(serverKey, broker)
		c.Set(clientKey, client)
		once.Do(func() {
			// this is some really hacky stuff
			// turns out I need to do some refactoring
			// don't judge!
			// will fix in 0.6 release
			ctx := c.Copy()
			client.Connect(
				stomp.WithCredentials("x-token", secret),
			)
			client.Subscribe("/queue/updates", stomp.HandlerFunc(func(m *stomp.Message) {
				go handlers.HandleUpdate(ctx, m.Copy())
			}))
		})
	}
}
```

agent连接server

src/github.com/drone/drone/drone/agent/agent.go
``` go
func start(c *cli.Context) {
	var client *stomp.Client
	for {
		// dial the drone server to establish a TCP connection.
		client, err = stomp.Dial(server)
		if err != nil {
			logger.Warningf("connection failed, retry in %v. %s", backoff, err)
			<-time.After(backoff)
			continue
		}
		opts := []stomp.MessageOption{
			stomp.WithCredentials("x-token", accessToken),
		}

		// initialize the stomp session and authenticate.
		if err = client.Connect(opts...); err != nil {
			logger.Warningf("session failed, retry in %v. %s", backoff, err)
			<-time.After(backoff)
			continue
		}
```

服务器收到请求之后的处理逻辑很简单，直接转发到mq

src/github.com/drone/drone/server/broker.go
``` go
// Broker handles connections to the embedded message broker.
func Broker(c *gin.Context) {
	broker := c.MustGet("broker").(http.Handler)
	broker.ServeHTTP(c.Writer, c.Request)
}
```

agent发送消息在【src/github.com/drone/drone/drone/agent/exec.go】
``` go
func (r *pipeline) run(w *model.Work) {
	a := agent.Agent{
		Update:    agent.NewClientUpdater(r.drone),
		Logger:    agent.NewClientLogger(r.drone, w.Job.ID, r.config.logs),
		Engine:    engine,
		Timeout:   r.config.timeout,
		Platform:  r.config.platform,
		Namespace: r.config.namespace,
		Escalate:  r.config.privileged,
		Extension: r.config.extension,
		Pull:      r.config.pull,
	}
```

src/github.com/drone/drone/agent/updater.go

``` go
// NewClientUpdater returns an updater that sends updated build details
// to the drone server.
func NewClientUpdater(client *stomp.Client) UpdateFunc {
	return func(w *model.Work) {
		err := client.SendJSON("/queue/updates", w)
		if err != nil {
			logger.Warningf("Error updating %s/%s#%d.%d. %s",
				w.Repo.Owner, w.Repo.Name, w.Build.Number, w.Job.Number, err)
		}
		if w.Job.Status != model.StatusRunning {
			var dest = fmt.Sprintf("/topic/logs.%d", w.Job.ID)
			var opts = []stomp.MessageOption{
				stomp.WithHeader("eof", "true"),
				stomp.WithRetain("all"),
			}

			if err := client.Send(dest, []byte("eof"), opts...); err != nil {
				logger.Warningf("Error sending eof %s/%s#%d.%d. %s",
					w.Repo.Owner, w.Repo.Name, w.Build.Number, w.Job.Number, err)
			}
		}
	}
}
```

# Cache
drone实现了一个类似于redis，但是集成在一起的缓存器，代码在【src/github.com/drone/drone/cache】
