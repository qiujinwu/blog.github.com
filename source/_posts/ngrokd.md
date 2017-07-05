---
title: Ngrok服务端源码学习笔记
date: 2017-7-5 19:32:49
tags:
 - golang
 - ngrok
categories:
 - 源码学习
---

关于ngrok的使用，参考<http://blog.qiujinwu.com/2017/02/13/ngrok-reverse-proxy/>

源码地址 <https://github.com/inconshreveable/ngrok>，我fork一份在<https://github.com/qjw/ngrok>,代码相对路径**src/github.com/qjw/ngrok/src/ngrok/server/**。

main入口在**src/github.com/qjw/ngrok/src/ngrok/main**,分成客户端和服务端/ngrokd/ngrokd.go、/ngrok/ngrok.go。

``` bash
.
├── cli.go  命令行相关
├── control.go 客户端和服务端控制连接逻辑
├── http.go 处理来自公网的http（s）的请求逻辑
├── main.go 入口
├── metrics.go 一些统计相关的东西
├── registry.go  存储全局对象的"容器"
├── tls.go tls加密相关
└── tunnel.go 客户端和服务端隧道逻辑
```

# 术语
在了解ngrok服务器原理之前，有几个术语需要区分
1. controller，控制器，每个客户端对应一个controller，并且会绑定一条tcp连接，默认使用tls加密，代码逻辑control.go，conroller用于传输各种控制指令
2. tunnel，隧道，一个客户端到服务端有多个隧道，每个隧道有个**TYPE**，例如http、https、tcp，以及**URL**，例如test.ngrok.qiujinwu.com【*假设ngrok.qiujinwu.com绑定到了服务器*】。在服务端代码中，tunnel是个虚拟的实体，并没有绑定的tcp连接。当收到来自公网的请求时，会根据隧道url来匹配客户端
3. proxy，表示客户端到服务端的数据链路，根据外网请求的多少，客户端到服务端会有多个proxy
4. listener，tcp服务器，默认会开启（http/https/tunnel三个tcp服务器，tcp类型不明确）

## message
在controller中，会发送各种控制指令，这些指令定义在**src/github.com/qjw/ngrok/src/ngrok/msg/msg.go**，大体而言可以分为三类
1. 控制指令，用于客户端连接服务器的协商报文
2. proxy指令，用于服务器请求新的数据链路（**由于客户端属于内网，ngrok服务器无法主动建立到客户端的连接，所以服务器会先走控制连接通知客户端，让它发起数据连接请求**）
3. 心跳

### 序列化
序列化比较简单，把报文序列化成字符串，并且把报文名称放在最前面，代码**src/github.com/qjw/ngrok/src/ngrok/msg/pack.go**，例如
``` json
{"Type":"RegProxy","Payload":{"ClientId":"8c57f5cfd5b30dc3215f740f2ad72539"}}
```

src/github.com/qjw/ngrok/src/ngrok/msg/conn.go有一些从tcp连接（反）序列化的工具函数

# 全局对象
``` go
// GLOBALS
var (
	// 所有的tunnel
	tunnelRegistry  *TunnelRegistry
	// 所有的controller
	controlRegistry *ControlRegistry

	// 参数
	opts	  *Options
	// 所有的tcp服务器
	listeners map[string]*conn.Listener
)
```
``` go
// ControlRegistry maps a client ID to Control structures
type ControlRegistry struct {
	controls map[string]*Control
	log.Logger
	sync.RWMutex
}
// TunnelRegistry maps a tunnel URL to Tunnel structures
type TunnelRegistry struct {
	tunnels  map[string]*Tunnel
	affinity *cache.LRUCache
	log.Logger
	sync.RWMutex
}
```
``` go
type Tunnel struct {
	// request that opened the tunnel
	req *msg.ReqTunnel

	// ...
	// 关联到control
	ctl *Control
	// ...
}
```

# 连接建立
在服务器接收请求之前，会新建一个listenr，这个对象对tcp服务器进行了封装
## Listener
src/github.com/qjw/ngrok/src/ngrok/conn/conn.go
``` go
type Listener struct {
	net.Addr
	// 将请求accept的新tcp连接放入channel
	Conns chan *loggedConn
}

type loggedConn struct {
	tcp *net.TCPConn
	net.Conn
	log.Logger
	id  int32
	typ string
}
```
将net.TCPConn包装成loggedConn，用于区分日志，设置type，id等
``` go
func wrapConn(conn net.Conn, typ string) *loggedConn {
	switch c := conn.(type) {
	case *vhost.HTTPConn:
		wrapped := c.Conn.(*loggedConn)
		return &loggedConn{wrapped.tcp, conn, wrapped.Logger, wrapped.id, wrapped.typ}
	case *loggedConn:
		return c
	case *net.TCPConn:
		wrapped := &loggedConn{c, conn, log.NewPrefixLogger(), rand.Int31(), typ}
		wrapped.AddLogPrefix(wrapped.Id())
		return wrapped
	}

	return nil
}
```
``` go
func Listen(addr, typ string, tlsCfg *tls.Config) (l *Listener, err error) {
	// listen for incoming connections
	listener, err := net.Listen("tcp", addr)
	// 。。。
	l = &Listener{
		Addr:  listener.Addr(),
		Conns: make(chan *loggedConn),
	}

	go func() {
		for {
			rawConn, err := listener.Accept()
			c := wrapConn(rawConn, typ)
			// 新的连接放入channel
			l.Conns <- c
		}
	}()
	return
}
```

## 接收请求
``` go
func tunnelListener(addr string, tlsConfig *tls.Config) {
	// 建立tcp服务器
	listener, err := conn.Listen(addr, "tun", tlsConfig)
	if err != nil {
		panic(err)
	}

	// 从channel中等待新的请求到来
	for c := range listener.Conns {
		// 每个连接用新的goroutune
		go func(tunnelConn conn.Conn) {
			defer func() {
				// 自动处理异常
				if r := recover(); r != nil {
					tunnelConn.Info("tunnelListener failed with error %v: %s", r, debug.Stack())
				}
			}()

			// 读取消息
			var rawMsg msg.Message
			if rawMsg, err = msg.ReadMsg(tunnelConn); err != nil {
				return
			}

			switch m := rawMsg.(type) {
			case *msg.Auth:
				// 建立控制连接（controller）
				NewControl(tunnelConn, m)

			case *msg.RegProxy:
				// 建立数据连接
				NewProxy(tunnelConn, m)

			default:
				tunnelConn.Close()
			}
		}(c)
	}
}
```

一个典型的请求报文
``` json
{
  "Type": "Auth",
  "Payload": {
	"Version": "2",
	"MmVersion": "1.7",
	"User": "",
	"Password": "",
	"OS": "linux",
	"Arch": "amd64",
	"ClientId": "8c57f5cfd5b30dc3215f740f2ad72539"
  }
}
```
响应
``` json
{
  "Type": "AuthResp",
  "Payload": {
	"Version": "2",
	"MmVersion": "1.7",
	"ClientId": "8c57f5cfd5b30dc3215f740f2ad72539",
	"Error": ""
  }
}
```

## Control
``` go
func NewControl(ctlConn conn.Conn, authMsg *msg.Auth) {
	var err error

	// create the object
	c := &Control{
		auth:			authMsg,
		conn:			ctlConn,
		out:			 make(chan msg.Message),
		in:			  make(chan msg.Message),
		proxies:		 make(chan conn.Conn, 10),
		lastPing:		time.Now(),
		writerShutdown:  util.NewShutdown(),
		readerShutdown:  util.NewShutdown(),
		managerShutdown: util.NewShutdown(),
		shutdown:		util.NewShutdown(),
	}

	// 设置属性
	ctlConn.SetType("ctl")
	ctlConn.AddLogPrefix(c.id)

	// 版本判断
	if authMsg.Version != version.Proto {
		failAuth(fmt.Errorf("Incompatible versions. Server %s, client %s. Download a new version at http://ngrok.com", version.MajorMinor(), authMsg.Version))
		return
	}

	// 新增/更新control到全局Registry
	if replaced := controlRegistry.(c.id, c); replaced != nil {
		// 等待旧的完全关闭
		replaced.shutdown.WaitComplete()
	}

	// 新的goruntune监听写（需要最先开启，以便回复连接请求报文）
	go c.writer()

	// 回复连接建立确认报文
	c.out <- &msg.AuthResp{
		Version:   version.Proto,
		MmVersion: version.MajorMinor(),
		ClientId:  c.id,
	}

	// 预先申请一个proxy连接
	c.out <- &msg.ReqProxy{}

	// 一堆其他的后台goroutune
	go c.manager()
	go c.reader()
	go c.stopper()
}
```
``` go
// 发送控制消息
func (c *Control) writer() {
	for m := range c.out {
		if err := msg.WriteMsg(c.conn, m); err != nil {
			panic(err)
		}
	}
}
```
``` go
// 接收控制消息
func (c *Control) reader() {
	for {
		if msg, err := msg.ReadMsg(c.conn); err != nil {
			if err == io.EOF {
				c.conn.Info("EOF")
				return
			}
		} else {
			// 推送到c.in channel中
			c.in <- msg
		}
	}
}
```
``` go
func (c *Control) manager() {
	// 检查control超时
	reap := time.NewTicker(connReapInterval)

	for {
		select {
		case <-reap.C:
			// 检查是否超时
			if time.Since(c.lastPing) > pingTimeoutInterval {
				c.conn.Info("Lost heartbeat")
				c.shutdown.Begin()
			}
		case mRaw, ok := <-c.in:
			// 是否有新的消息
			if !ok {
				return
			}

			switch m := mRaw.(type) {
			case *msg.ReqTunnel:
				// 客户端注册一个新的tunnel
				c.registerTunnel(m)

			case *msg.Ping:
				// 回复心跳
				c.lastPing = time.Now()
				c.out <- &msg.Pong{}
			}
		}
	}
}
```
注册controller
``` go
func (r *ControlRegistry) Add(clientId string, ctl *Control) (oldCtl *Control) {
	r.Lock()
	defer r.Unlock()

	oldCtl = r.controls[clientId]
	if oldCtl != nil {
		oldCtl.Replaced(ctl)
	}

	r.controls[clientId] = ctl
	r.Info("Registered control with id %s", clientId)
	return
}
```

### 退出流程
**退出流程可以考虑用<https://golang.org/pkg/context/>简化下**，这里用到了一个util.Shutdown的工具库（src/github.com/qjw/ngrok/src/ngrok/util/shutdown.go）。
``` go
type Control struct {
	// synchronizer for controlled shutdown of writer()
	writerShutdown *util.Shutdown

	// synchronizer for controlled shutdown of reader()
	readerShutdown *util.Shutdown

	// synchronizer for controlled shutdown of manager()
	managerShutdown *util.Shutdown

	// synchronizer for controller shutdown of entire Control
	shutdown *util.Shutdown
}
```
``` go

func (c *Control) reader() {
	// kill everything if the reader stops
	defer c.shutdown.Begin()
	// notify that we're done
	defer c.readerShutdown.Complete()
}
```
``` go
func (c *Control) writer() {
	// kill everything if the writer() stops
	defer c.shutdown.Begin()

	// notify that we've flushed all messages
	defer c.writerShutdown.Complete()
}
```
``` go
func (c *Control) manager() {
	// kill everything if the control manager stops
	defer c.shutdown.Begin()

	// notify that manager() has shutdown
	defer c.managerShutdown.Complete()
}
```
有个专门的goruntune来监听退出
``` go
func (c *Control) stopper() {
	// 等待
	c.shutdown.WaitBegin()

	// 注销controller
	controlRegistry.Del(c.id)

	// 等待各种子goruntune注销
	// close会触发其他的goruntune退出
	close(c.in)
	c.managerShutdown.WaitComplete()

	// shutdown writer()
	close(c.out)
	c.writerShutdown.WaitComplete()

	// 关闭空置连接
	c.conn.Close()

	// 关闭各种tunnel
	for _, t := range c.tunnels {
		// 调用shutdown，
		t.Shutdown()
	}

	// 关闭各种proxy连接
	close(c.proxies)
	for p := range c.proxies {
		p.Close()
	}

	// 最终关闭
	c.shutdown.Complete()
	c.conn.Info("Shutdown complete")
}
```

## Tunnel注册
``` go
// Register a new tunnel on this control connection
func (c *Control) registerTunnel(rawTunnelReq *msg.ReqTunnel) {
	// 若有多个tunnel，可以用一个报文一次性注册
	for _, proto := range strings.Split(rawTunnelReq.Protocol, "+") {
		tunnelReq := *rawTunnelReq
		tunnelReq.Protocol = proto

		c.conn.Debug("Registering new tunnel")
		t, err := NewTunnel(&tunnelReq, c)
		if err != nil {
			// 回复注册失败确认
			c.out <- &msg.NewTunnel{Error: err.Error()}
			return
		}

		// 注册到controller
		c.tunnels = append(c.tunnels, t)

		// 回复注册成功确认
		c.out <- &msg.NewTunnel{
			Url:	  t.url,
			Protocol: proto,
			ReqId:	rawTunnelReq.ReqId,
		}

		rawTunnelReq.Hostname = strings.Replace(t.url, proto+"://", "", 1)
	}
}
```
一个典型的注册报文
``` json
{
  "Type": "ReqTunnel",
  "Payload": {
	"ReqId": "dd1819bd088d7675",
	"Protocol": "http",
	"Hostname": "",
	"Subdomain": "qjw",
	"HttpAuth": "",
	"RemotePort": 0
  }
}
```
响应
``` json
{
  "Type": "NewTunnel",
  "Payload": {
	"ReqId": "dd1819bd088d7675",
	"Url": "http://qjw.ngrok.com:10080",
	"Protocol": "http",
	"Error": ""
  }
}
```

``` go
// Create a new tunnel from a registration message received
// on a control channel
func NewTunnel(m *msg.ReqTunnel, ctl *Control) (t *Tunnel, err error) {
	t = &Tunnel{
		req:	m,
		start:  time.Now(),
		ctl:	ctl,
		Logger: log.NewPrefixLogger(),
	}

	proto := t.req.Protocol
	switch proto {
	case "tcp":
		// ...
		return

	case "http", "https":
		l, ok := listeners[proto]
		// 注册vhost，之所以v，是因为多个url共享同一个端口，类似于nginx的server
		if err = registerVhost(t, proto, l.Addr.(*net.TCPAddr).Port); err != nil {
			return
		}

	default:
	}
	t.AddLogPrefix(t.Id())
	return
}
```
这里主要确认vhost的参数，最重要的host，port，这对于公网连接的请求路由至关重要
``` go
var defaultPortMap = map[string]int{
	"http":  80,
	"https": 443,
	"smtp":  25,
}

// Common functionality for registering virtually hosted protocols
func registerVhost(t *Tunnel, protocol string, servingPort int) (err error) {
	vhost := os.Getenv("VHOST")
	if vhost == "" {
		vhost = fmt.Sprintf("%s:%d", opts.domain, servingPort)
	}

	// Canonicalize virtual host by removing default port (e.g. :80 on HTTP)
	defaultPort, ok := defaultPortMap[protocol]
	if !ok {
		return fmt.Errorf("Couldn't find default port for protocol %s", protocol)
	}

	// 移除默认的端口（比如80可以忽略，81就必须明确地出现在连接中）
	defaultPortSuffix := fmt.Sprintf(":%d", defaultPort)
	if strings.HasSuffix(vhost, defaultPortSuffix) {
		vhost = vhost[0 : len(vhost)-len(defaultPortSuffix)]
	}

	// Canonicalize by always using lower-case
	vhost = strings.ToLower(vhost)

	// 从请求中获取host
	hostname := strings.ToLower(strings.TrimSpace(t.req.Hostname))
	if hostname != "" {
		t.url = fmt.Sprintf("%s://%s", protocol, hostname)
		// 注册tunnel
		return tunnelRegistry.Register(t.url, t)
	}

	// 未指定host，指定了subdomain，就通过服务器启动参数domain来拼装host
	subdomain := strings.ToLower(strings.TrimSpace(t.req.Subdomain))
	if subdomain != "" {
		t.url = fmt.Sprintf("%s://%s.%s", protocol, subdomain, vhost)
		// 注册tunnel
		return tunnelRegistry.Register(t.url, t)
	}

	// 随机生成一个url
	t.url, err = tunnelRegistry.RegisterRepeat(func() string {
		return fmt.Sprintf("%s://%x.%s", protocol, rand.Int31(), vhost)
	}, t)

	return
}
```
``` go
func (t *Tunnel) Shutdown() {
	// 取消注册
	tunnelRegistry.Del(t.url)
}
```

## Proxy
在连接建立之后，服务器就会预先请求一个proxy
``` json
{"Type":"ReqProxy","Payload":{}}
```
这个clientid非常重要，用于关联到controller
``` json
{"Type":"RegProxy","Payload":{"ClientId":"8c57f5cfd5b30dc3215f740f2ad72539"}}
```

``` go
func NewControl(ctlConn conn.Conn, authMsg *msg.Auth) {
	// 预先申请一个proxy
	c.out <- &msg.ReqProxy{}
}
```
``` go
func tunnelListener(addr string, tlsConfig *tls.Config) {
	for c := range listener.Conns {
		go func(tunnelConn conn.Conn) {
			switch m := rawMsg.(type) {
			case *msg.RegProxy:
				// 新增一个proxy
				NewProxy(tunnelConn, m)
			}
		}(c)
	}
}
```

proxy并没有全局的对象来注册，而是简单地关联到controller（通过之前的clientid），具体就是通过一个带缓存的channel
``` go
type Control struct {
	// proxy connections
	proxies chan conn.Conn
}

func NewControl(ctlConn conn.Conn, authMsg *msg.Auth) {
	c := &Control{
		// 10个元素的channel
		proxies:		 make(chan conn.Conn, 10),
	}
}
```

``` go
func NewProxy(pxyConn conn.Conn, regPxy *msg.RegProxy) {
	pxyConn.SetType("pxy")

	// 查询controller
	ctl := controlRegistry.Get(regPxy.ClientId)

	// 注册
	ctl.RegisterProxy(pxyConn)
}
```

当需要获取proxy时，就直接调用下面的函数
``` go
func (c *Control) GetProxy() (proxyConn conn.Conn, err error) {
	var ok bool

	// get a proxy connection from the pool
	select {
	// 直接从channel中获取
	case proxyConn, ok = <-c.proxies:
		if !ok {
			err = fmt.Errorf("No proxy connections available, control is closing")
			return
		}
	default:
		//  没有的话，立即请求客户端
		if err = util.PanicToError(func() { c.out <- &msg.ReqProxy{} }); err != nil {
			return
		}

		// 继续从channle中获取
		select {
		case proxyConn, ok = <-c.proxies:
			if !ok {
				err = fmt.Errorf("No proxy connections available, control is closing")
				return
			}

		case <-time.After(pingTimeoutInterval):
			err = fmt.Errorf("Timeout trying to get proxy connection")
			return
		}
	}
	return
}
```

# 处理公网请求

``` go
func Main() {
	// listen for http
	if opts.httpAddr != "" {
		listeners["http"] = startHttpListener(opts.httpAddr, nil)
	}
}
```
``` go
// Listens for new http(s) connections from the public internet
func startHttpListener(addr string, tlsCfg *tls.Config) (listener *conn.Listener) {
	// 创建服务器
	var err error
	if listener, err = conn.Listen(addr, "pub", tlsCfg); err != nil {
		panic(err)
	}

	proto := "http"
	if tlsCfg != nil {
		proto = "https"
	}

	log.Info("Listening for public %s connections on %v", proto, listener.Addr.String())
	go func() {
		for conn := range listener.Conns {
			// 每个连接都会调用下面的goruntune
			go httpHandler(conn, proto)
		}
	}()

	return
}
```
``` go
func httpHandler(c conn.Conn, proto string) {

	// 获取Http头
	vhostConn, err := vhost.HTTP(c)
	if err != nil {
		c.Warn("Failed to read valid %s request: %v", proto, err)
		c.Write([]byte(BadRequest))
		return
	}

	// 获取http参数
	host := strings.ToLower(vhostConn.Host())
	auth := vhostConn.Request.Header.Get("Authorization")

	// done reading mux data, free up the request memory
	vhostConn.Free()

	// We need to read from the vhost conn now since it mucked around reading the stream
	c = conn.Wrap(vhostConn, "pub")

	// 从全局的Registry中查找tunnel
	tunnel := tunnelRegistry.Get(fmt.Sprintf("%s://%s", proto, host))
	if tunnel == nil {
		c.Info("No tunnel found for hostname %s", host)
		c.Write([]byte(fmt.Sprintf(NotFound, len(host)+18, host)))
		return
	}

	// 检查认证
	if tunnel.req.HttpAuth != "" && auth != tunnel.req.HttpAuth {
		c.Info("Authentication failed: %s", auth)
		c.Write([]byte(NotAuthorized))
		return
	}

	// 数据交换
	tunnel.HandlePublicConnection(c)
}
```

## Http头
在数据交换过程中，为了作请求路由，必须先从路由中解析Http头（不考虑tcp tunnel），然后根据http头来作数据路由，而读出来的头，也必须原原本本的下发到客户端。

``` go
const (
	initVhostBufSize = 1024 // allocate 1 KB up front to try to avoid resizing
)

type sharedConn struct {
	sync.Mutex
	net.Conn			   // the raw connection
	vhostBuf *bytes.Buffer // all of the initial data that has to be read in order to vhost a connection is saved here
}

func newShared(conn net.Conn) (*sharedConn, io.Reader) {
	c := &sharedConn{
		Conn:	 conn,
		// 分配一块内存，用于存储http头，以便原原本本的下发到客户端
		vhostBuf: bytes.NewBuffer(make([]byte, 0, initVhostBufSize)),
	}

	// 当从conn读取数据后，复制一份到vhostBuf
	return c, io.TeeReader(conn, c.vhostBuf)
}

func (c *sharedConn) Read(p []byte) (n int, err error) {
	c.Lock()
	// 已经读取到内存中的数据已经发送完
	if c.vhostBuf == nil {
		c.Unlock()
		return c.Conn.Read(p)
	}

	// 优先从buf中读取
	n, err = c.vhostBuf.Read(p)

	// end of the request buffer
	if err == io.EOF {
		// let the request buffer get garbage collected
		// and make sure we don't read from it again
		c.vhostBuf = nil

		// 读了一半继续从con中读取
		// continue reading from the connection
		var n2 int
		n2, err = c.Conn.Read(p[n:])

		// update total read
		n += n2
	}
	c.Unlock()
	return
}
```

vhost.HTTP
``` go
func HTTP(conn net.Conn) (httpConn *HTTPConn, err error) {
	// 创建一个tee的conn
	c, rd := newShared(conn)

	// 解析Http头，不得不说go的系统库很牛X
	httpConn = &HTTPConn{sharedConn: c}
	if httpConn.Request, err = http.ReadRequest(bufio.NewReader(rd)); err != nil {
		return
	}

	// body不需要
	httpConn.Request.Body.Close()
	return
}
```

由于有了sharedConn这一层封装，在数据交换时，完全可以不考虑，一部分数据已经为了解析http头而实现读取出来过的细节

## 数据交换
``` go
func (t *Tunnel) HandlePublicConnection(publicConn conn.Conn) {
	for i := 0; i < (2 * proxyMaxPoolSize); i++ {
		// 获取一个proxy
		if proxyConn, err = t.ctl.GetProxy(); err != nil {
			t.Warn("Failed to get proxy connection: %v", err)
			return
		}
		// 自动关闭和设置属性
		defer proxyConn.Close()
		t.Info("Got proxy connection %s", proxyConn.Id())
		proxyConn.AddLogPrefix(t.Id())

		// 发送数据，提示开始传输数据
		startPxyMsg := &msg.StartProxy{
			Url:		t.url,
			ClientAddr: publicConn.RemoteAddr().String(),
		}
		if err = msg.WriteMsg(proxyConn, startPxyMsg); err != nil {
			proxyConn.Warn("Failed to write StartProxyMessage: %v, attempt %d", err, i)
			proxyConn.Close()
		} else {
			// success
			break
		}
	}

	// 立即申请一个新的proxy
	util.PanicToError(func() { t.ctl.out <- &msg.ReqProxy{} })

	// 交换数据
	bytesIn, bytesOut := conn.Join(publicConn, proxyConn)
}
```

通过一个join实现双向的数据交换，**这个地方可以做一些部分优化，直接从内核到内核传数据，参考[sendfile](https://linux.die.net/man/2/sendfile)、[splice](https://linux.die.net/man/2/splice)**
``` go
func Join(c Conn, c2 Conn) (int64, int64) {
	var wait sync.WaitGroup

	pipe := func(to Conn, from Conn, bytesCopied *int64) {
		defer to.Close()
		defer from.Close()
		defer wait.Done()

		var err error
		// 这个地方可以优化，在内部有一个应用层的buf作中转
		*bytesCopied, err = io.Copy(to, from)
		if err != nil {
			from.Warn("Copied %d bytes to %s before failing with error %v", *bytesCopied, to.Id(), err)
		} else {
			from.Debug("Copied %d bytes to %s", *bytesCopied, to.Id())
		}
	}

	wait.Add(2)
	var fromBytes, toBytes int64
	// 开启两个goruntune来实现双向交换
	go pipe(c, c2, &fromBytes)
	go pipe(c2, c, &toBytes)
	c.Info("Joined with connection %s", c2.Id())
	wait.Wait()
	return fromBytes, toBytes
}
```

# tls加密
对于tls支持，go系统库支持的非常好，参考<http://colobu.com/2016/06/07/simple-golang-tls-examples/>

ngrok有点讨巧，用系统库的工具函数在tcp裸连接做了一层包装
``` go
func Listen(addr, typ string, tlsCfg *tls.Config) (l *Listener, err error) {
	// 监听tcp端口
	listener, err := net.Listen("tcp", addr)

	// 声明对象
	l = &Listener{
		Addr:  listener.Addr(),
		Conns: make(chan *loggedConn),
	}

	go func() {
		for {
			rawConn, err := listener.Accept()

			// 处理新连接
			c := wrapConn(rawConn, typ)
			// 若指定了tls配置（https tunnel必需）
			if tlsCfg != nil {
				// 直接将原来的裸tcp conn替换成被tls包装过的conn，所有的read/write会被tls层先做加工再下发
				c.Conn = tls.Server(c.Conn, tlsCfg)
			}
			l.Conns <- c
		}
	}()
	return
}
```

## Client
在ngrok客户端也是同样的逻辑
``` go
func Dial(addr, typ string, tlsCfg *tls.Config) (conn *loggedConn, err error) {
	var rawConn net.Conn
	if rawConn, err = net.Dial("tcp", addr); err != nil {
		return
	}

	conn = wrapConn(rawConn, typ)
	conn.Debug("New connection to: %v", rawConn.RemoteAddr())

	// 若指定了tls配置（https tunnel必需）
	if tlsCfg != nil {
		conn.StartTLS(tlsCfg)
	}
	return
}
func (c *loggedConn) StartTLS(tlsCfg *tls.Config) {
	// 直接将原来的裸tcp conn替换成被tls包装过的conn，所有的read/write会被tls层先做加工再下发
	c.Conn = tls.Client(c.Conn, tlsCfg)
}
```

