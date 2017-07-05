---
title: Drone学习笔记4
date: 2017-3-8 11:17:22
tags:
 - drone
 - golang
 - 持续集成
categories:
 - 源码学习
---

找到一个编译成功过的resp【qjw/test】，点击restart

# 发起编译请求

前端发送POST请求【/api/repos/qjw/test/builds/2】，2是build的id。同时发起ws Get请求【ws://test.ycy.qiujinwu.com/ws/logs/qjw/test/2/1】，2是build的id，1是job的id。

``` go
ws.GET("/logs/:owner/:name/:build/:number",
			session.SetRepo(),
			session.SetPerm(),
			session.MustPull,
			server.LogStream,
		)

func SetRepo() gin.HandlerFunc {
	return func(c *gin.Context) {
		var (
			owner = c.Param("owner")
			name  = c.Param("name")
			user  = User(c)
		)

// LogStream streams the build log output to the client.
func LogStream(c *gin.Context) {
	repo := session.Repo(c)
	buildn, _ := strconv.Atoi(c.Param("build"))
	jobn, _ := strconv.Atoi(c.Param("number"))
```

发起编译的请求最终会跑到

src/github.com/drone/drone/server/build.go
``` go
func PostBuild(c *gin.Context) {
	// 获得后端实例
	remote_ := remote.FromContext(c)

	// 从数据库中获取的resp对象，通过前置的middleware完成了查询
	repo := session.Repo(c)
	fork := c.DefaultQuery("fork", "false")

	// 获得当前用户
	user, err := store.GetUser(c, repo.UserID)
	if err != nil {
		log.Errorf("failure to find repo owner %s. %s", repo.FullName, err)
		c.AbortWithError(500, err)
		return
	}

	// 获得build number
	build, err := store.GetBuildNumber(c, repo, num)
	if err != nil {
		log.Errorf("failure to get build %d. %s", num, err)
		c.AbortWithError(404, err)
		return
	}

	// 拉取.drone.yml文件
	// fetch the .drone.yml file from the database
	config := ToConfig(c)
	raw, err := remote_.File(user, repo, build, config.Yaml)
	if err != nil {
		log.Errorf("failure to get build config for %s. %s", repo.FullName, err)
		c.AbortWithError(404, err)
		return
	}

	// 检查是否有签名文件
	// Fetch secrets file but don't exit on error as it's optional
	sec, err := remote_.File(user, repo, build, config.Shasum)
	if err != nil {
		log.Debugf("cannot find build secrets for %s. %s", repo.FullName, err)
	}

	// 验证签名
	signature, err := jose.ParseSigned(string(sec))
	if err != nil {
		log.Debugf("cannot parse .drone.yml.sig file. %s", err)
	} else if len(sec) == 0 {
		log.Debugf("cannot parse .drone.yml.sig file. empty file")
	} else {
		signed = true
		output, err := signature.Verify([]byte(repo.Hash))
		if err != nil {
			log.Debugf("cannot verify .drone.yml.sig file. %s", err)
		} else if string(output) != string(raw) {
			log.Debugf("cannot verify .drone.yml.sig file. no match. %q <> %q", string(output), string(raw))
		} else {
			verified = true
		}
	}

	// 发送请求到tast list
	client := stomp.MustFromContext(c)
	client.SendJSON("/topic/events", model.Event{
		Type:  model.Enqueued,
		Repo:  *repo,
		Build: *build,
	},
		stomp.WithHeader("repo", repo.FullName),
		stomp.WithHeader("private", strconv.FormatBool(repo.IsPrivate)),
	)

	for _, job := range jobs {
		broker, _ := stomp.FromContext(c)
		// 发送请求到agent
		broker.SendJSON("/queue/pending", &model.Work{
			Signed:    signed,
			Verified:  verified,
			User:      user,
			Repo:      repo,
			Build:     build,
			BuildLast: last,
			Job:       job,
			Netrc:     netrc,
			Yaml:      string(raw),
			Secrets:   secs,
			System:    &model.System{Link: httputil.GetURL(c.Request)},
		},
			stomp.WithHeader(
				"platform",
				yaml.ParsePlatformDefault(raw, "linux/amd64"),
			),
			stomp.WithHeaders(
				yaml.ParseLabel(raw),
			),
		)
	}
}
```

## .drone.yml签名
签名通过cli完成

src/github.com/drone/drone/drone/sign.go
``` go
var signCmd = cli.Command{
	Name:  "sign",
	Usage: "creates a secure yaml file",
	Action: func(c *cli.Context) {
		if err := sign(c); err != nil {
			log.Fatalln(err)
		}
	},
	Flags: []cli.Flag{
		cli.StringFlag{
			Name:  "in",
			Usage: "input file",
			Value: ".drone.yml",
		},
		cli.StringFlag{
			Name:  "out",
			Usage: "output file signature",
			Value: ".drone.yml.sig",
		},
	},
}
```

最终调用接口
``` bash
repo.POST("/sign", session.MustPush, server.Sign)
```

# agent接收

src/github.com/drone/drone/drone/agent/agent.go
``` go
func start(c *cli.Context) {
	// 初始化全局的docker实例
	docker, err := dockerclient.NewDockerClient(c.String("docker-host"), tls)
	if err != nil {
		logrus.Fatal(err)
	}

	var client *stomp.Client

	// 收到请求的回调
	handler := func(m *stomp.Message) {
		running.Add(1)
		defer func() {
			running.Done()
			client.Ack(m.Ack)
		}()

		r := pipeline{
			drone:  client,
			docker: docker,
			config: config{
				platform:   c.String("docker-os") + "/" + c.String("docker-arch"),
				timeout:    c.Duration("timeout"),
				namespace:  c.String("namespace"),
				privileged: c.StringSlice("privileged"),
				pull:       c.BoolT("pull"),
				logs:       int64(c.Int("max-log-size")) * 1000000,
				extension:  c.StringSlice("extension"),
			},
		}

		work := new(model.Work)
		m.Unmarshal(work)
		r.run(work)
	}

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

		// 连接mq
		// initialize the stomp session and authenticate.
		if err = client.Connect(opts...); err != nil {
			logger.Warningf("session failed, retry in %v. %s", backoff, err)
			<-time.After(backoff)
			continue
		}

        // 订阅topic
		// subscribe to the pending build queue.
		client.Subscribe("/queue/pending", stomp.HandlerFunc(func(m *stomp.Message) {
			go handler(m) // HACK until we a channel based Subscribe implementation
		}), opts...)
```

在上面开始有了pipeline的概念，参见.drone.yml的pipeline定义。

在【m.Unmarshal(work)】会反序列化MQ消息到Work struct
``` go
// Work represents an item for work to be
// processed by a worker.
type Work struct {
	Signed    bool      `json:"signed"`
	Verified  bool      `json:"verified"`
	Yaml      string    `json:"config"`
	YamlEnc   string    `json:"secret"`
	Repo      *Repo     `json:"repo"`
	Build     *Build    `json:"build"`
	BuildLast *Build    `json:"build_last"`
	Job       *Job      `json:"job"`
	Netrc     *Netrc    `json:"netrc"`
	Keys      *Key      `json:"keys"`
	System    *System   `json:"system"`
	Secrets   []*Secret `json:"secrets"`
	User      *User     `json:"user"`
}
```

重点关注**Yaml**字段

## Docker client
drone 并没有直接使用的第三方开源的client，而是封装一个接口，代码实现在【src/github.com/drone/drone/build/docker】。在后端再使用第三方**[samalba/dockerclient](https://godoc.org/github.com/samalba/dockerclient)**

``` go
// Engine defines the container runtime engine.
type Engine interface {
	ContainerStart(*yaml.Container) (string, error)
	ContainerStop(string) error
	ContainerRemove(string) error
	ContainerWait(string) (*State, error)
	ContainerLogs(string) (io.ReadCloser, error)
}
```

# Topic
1. /queue/updates server接收来自agent的执行结果
2. /topic/logs.%d server接收来自agent的编译实时日志
3. /topic/events 事件列表，用于刷新drone ui左边列表的任务，和agent无交互
4. /topic/cancel 向agent发送任务取消指令
5. /queue/pending 向agent发送编译指令，含1. 用户手动触发、2. git仓库回调触发比如push了新代码

/topic/logs 和 /topic/events最终会关联到两个ws接口，用于实时刷新web ui。

``` go
ws.GET("/feed", server.EventStream)
ws.GET("/logs/:owner/:name/:build/:number",
    session.SetRepo(),
    session.SetPerm(),
    session.MustPull,
    server.LogStream,
)
```


# Pipeline
接下来会创建一个Agent对象，并Run

src/github.com/drone/drone/drone/agent/exec.go
``` go
func (r *pipeline) run(w *model.Work) {
	// 创建cancel channel
	cancel := make(chan bool, 1)
    // 创建docker实例
	engine := docker.NewClient(r.docker)
	// 创建agent实例
	a := agent.Agent{
    	// 发送build日志
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


	// 支持取消编译事件
	// signal for canceling the build.
	sub, err := r.drone.Subscribe("/topic/cancel", stomp.HandlerFunc(cancelFunc))
	if err != nil {
		logrus.Errorf("Error subscribing to /topic/cancel. %s", err)
	}

	a.Run(w, cancel)

}
```

接着解释.drone.yml，并执行docker

src/github.com/drone/drone/agent/agent.go
``` go
func (a *Agent) Run(payload *model.Work, cancel <-chan bool) error {

	// 预处理
	spec, err := a.prep(payload)

	// Update是一个value为函数的回调值，用于更新任务状态，见agent.NewClientUpdater
	a.Update(payload)

    // 执行
	err = a.exec(spec, payload, cancel)

	a.Update(payload)
	return err
}
```

## 预处理
预处理有两个任务很关键

1. 使用配置的密文替换.drone.yml的变量，参见envsubst.Eval
2. 解析.drone.xml，保存到Config结构中，参见conf, err := yaml.ParseString(w.Yaml)

``` go
// Config represents the build configuration Yaml document.
type  struct {
	Image     string
	Build     *Build
	Workspace *Workspace
	Pipeline  []*Container
	Services  []*Container
	Volumes   []*Volume
	Networks  []*Network
}
```

``` go
func (a *Agent) prep(w *model.Work) (*yaml.Config, error) {

	envs := toEnv(w)
	envSecrets := map[string]string{}

	// list of secrets to interpolate in the yaml
	for _, secret := range w.Secrets {
		if (w.Verified || secret.SkipVerify) && secret.MatchEvent(w.Build.Event) {
			envSecrets[secret.Name] = secret.Value
		}
	}

	var err error
	w.Yaml, err = envsubst.Eval(w.Yaml, func(s string) string {
		env, ok := envSecrets[s]
		if !ok {
			env, _ = envs[s]
		}
		if strings.Contains(env, "\n") {
			env = fmt.Sprintf("%q", env)
		}
		return env
	})
	if err != nil {
		return nil, err
	}


	conf, err := yaml.ParseString(w.Yaml)
	if err != nil {
		return nil, err
	}

	return conf, nil
}
```

### 密文
drone数据库有一个专门的表sercrets用来存储这些字段，webui没有提供入口配置，需要使用cli，具体参见官方文档<http://readme.drone.io/cli/drone-secret-add/>。这些密文通过mq从server带过来，agent并不直接存储。

关于密文使用，参见官方<http://readme.drone.io/usage/secret-guide/>

具体的替换过程，使用函数envsubst.Eval，代码在<https://github.com/drone/drone/tree/master/vendor/github.com/drone/envsubst>，支持的模式如下：

```
Supported Functions:
    ${var^}
    ${var^^}
    ${var,}
    ${var,,}
    ${var:position}
    ${var:position:length}
    ${var#substring}
    ${var##substring}
    ${var%substring}
    ${var%%substring}
    ${var/substring/replacement}
    ${var//substring/replacement}
    ${var/#substring/replacement}
    ${var/%substring/replacement}
    ${#var}
    ${var=default}
    ${var:=default}
    ${var:-default}

Unsupported Functions:
    ${var-default}
    ${var+default}
    ${var:?default}
    ${var:+default}
```

## 解析.drone.yml
解析的入口在yaml.ParseString，代码实现在【src/github.com/drone/drone/yaml】，关于Config结构，我们重点关注以下两个字段
``` go
type  struct {
	Pipeline  []*Container
	Services  []*Container
}
```

他们最终都会存储为一个**有序**的链表，代码实现【src/github.com/drone/drone/yaml/container.go】，下面是最简单的.drone.yml

``` yaml
pipeline:
  build:
    image: golang
    commands:
      - go get
      - go build
      - go test

services:
  postgres:
    image: postgres:9.4.5
    environment:
      - POSTGRES_USER=myapp
```

所以.drone.yml支持哪些字段，看看【src/github.com/drone/drone/yaml】的对象定义即可，关注对象字段tag和UnmarshalYAML函数。
``` go
// Container defines a Docker container.
type Container struct {
	ID             string
	Name           string
	Image          string
	Build          string
	Pull           bool
	AuthConfig     Auth
	Detached       bool
	Disabled       bool
	Privileged     bool
	WorkingDir     string
	Environment    map[string]string
	Labels         map[string]string
	Entrypoint     []string
	Command        []string
	Commands       []string
	ExtraHosts     []string
	Volumes        []string
	VolumesFrom    []string
	Devices        []string
	Network        string
	DNS            []string
	DNSSearch      []string
	MemSwapLimit   int64
	MemLimit       int64
	ShmSize        int64
	CPUQuota       int64
	CPUShares      int64
	CPUSet         string
	OomKillDisable bool
	Constraints    Constraints

	Vargs map[string]interface{}
}
```

解析完.drone.xml之后，会自动在前面加上一个clone的容器

src/github.com/drone/drone/yaml/transform/clone.go

``` go
// Clone transforms the Yaml to include a clone step.
func Clone(c *yaml.Config, plugin string) error {
	switch plugin {
	case "", "git":
		plugin = "plugins/git:latest"
	case "hg":
		plugin = "plugins/hg:latest"
	}

	for _, p := range c.Pipeline {
		if p.Name == clone {
			if p.Image == "" {
				p.Image = plugin
			}
			return nil
		}
	}

	s := &yaml.Container{
		Image: plugin,
		Name:  clone,
	}

	c.Pipeline = append([]*yaml.Container{s}, c.Pipeline...)
	return nil
}
```

最后还会加上一个busybox的容器，我的理解这个容器的目的仅仅是为了在最开始挂载磁盘，以便维持pipeline各个容器的共享代码。
``` go
// lookup ambassador configuration by architecture and os
var lookupAmbassador = map[string]ambassador{
	"linux/amd64": {
		image:      "busybox:latest",
		entrypoint: []string{"/bin/sleep"},
		command:    []string{"86400"},
	},
	"linux/arm": {
		image:      "armhf/alpine:latest",
		entrypoint: []string{"/bin/sleep"},
		command:    []string{"86400"},
	},
}

// Pod transforms the containers in the Yaml to use Pod networking, where every
// container shares the localhost connection.
func Pod(c *yaml.Config, platform string) error {

	rand := base64.RawURLEncoding.EncodeToString(
		securecookie.GenerateRandomKey(8),
	)

	// choose the ambassador configuration based on os and architecture
	conf, ok := lookupAmbassador[platform]
	if !ok {
		conf = defaultAmbassador
	}

    for _, container := range containers {
		container.VolumesFrom = append(container.VolumesFrom, ambassador.ID)
		if container.Network == "" {
			container.Network = network
		}
	}
```

留意pod函数最后的for循环，**通过VolumesFrom实现了所有容器共享一个磁盘，从而共享代码等数据**


## 执行容器
src/github.com/drone/drone/build/config.go
``` go
// Pipeline creates a build Pipeline using the specific configuration for
// the given Yaml specification.
func (c *Config) Pipeline(spec *yaml.Config) *Pipeline {

	pipeline := Pipeline{
		engine: c.Engine,
		pipe:   make(chan *Line, c.Buffer),
		next:   make(chan error),
		done:   make(chan error),
	}

	var containers []*yaml.Container
	containers = append(containers, spec.Services...)
	containers = append(containers, spec.Pipeline...)

	for _, c := range containers {
		if c.Disabled {
			continue
		}
		next := &element{Container: c}
		if pipeline.head == nil {
			pipeline.head = next
			pipeline.tail = next
		} else {
			pipeline.tail.next = next
			pipeline.tail = next
		}
	}

	go func() {
		pipeline.next <- nil
	}()

	return &pipeline
}
```

``` go
func (a *Agent) exec(spec *yaml.Config, payload *model.Work, cancel <-chan bool) error {

	conf := build.Config{
		Engine: a.Engine,
		Buffer: 500,
	}

	pipeline := conf.Pipeline(spec)
	defer pipeline.Teardown()

	// setup the build environment
	if err := pipeline.Setup(); err != nil {
		return err
	}

	replacer := NewSecretReplacer(payload.Secrets)
	timeout := time.After(time.Duration(payload.Repo.Timeout) * time.Minute)

	for {
		select {
		case <-pipeline.Done():
			return pipeline.Err()
		case <-cancel:
			pipeline.Stop()
			return fmt.Errorf("termination request received, build cancelled")
		case <-timeout:
			pipeline.Stop()
			return fmt.Errorf("maximum time limit exceeded, build cancelled")
		case <-time.After(a.Timeout):
			pipeline.Stop()
			return fmt.Errorf("terminal inactive for %v, build cancelled", a.Timeout)
		case <-pipeline.Next():

			// TODO(bradrydzewski) this entire block of code should probably get
			// encapsulated in the pipeline.
			status := model.StatusSuccess
			if pipeline.Err() != nil {
				status = model.StatusFailure
			}
			// updates the build status passed into each container. I realize this is
			// a bit out of place and will work to resolve.
			pipeline.Head().Environment["DRONE_BUILD_STATUS"] = status

			if !pipeline.Head().Constraints.Match(
				a.Platform,
				payload.Build.Deploy,
				payload.Build.Event,
				payload.Build.Branch,
				status, payload.Job.Environment) { // TODO: fix this whole section

				pipeline.Skip()
			} else {
				pipeline.Exec()
			}
		case line := <-pipeline.Pipe():
			line.Out = replacer.Replace(line.Out)
			a.Logger(line)
		}
	}
}
```

结束之后会自动停止和删除容器，但不会删除镜像，参见[src/github.com/drone/drone/build/pipeline.go]
