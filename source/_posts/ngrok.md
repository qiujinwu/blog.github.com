---
title: Ngrok客户端源码学习笔记
date: 2017-7-5 22:29:10
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
├── cli.go #命令行相关
├── config.go #配置相关
├── controller.go # 控制器，用于管理view/网络/config/state等
├── main.go #入口
├── metrics.go #统计数据
├── model.go # 核心逻辑
├── mvc
│   ├── controller.go # controller interface
│   ├── model.go # interface
│   ├── state.go # interface和状态定义
│   └── view.go # interface
├── tls.go #tls加密
└── views  #view
    ├── term #终端view
    │   ├── area.go
    │   ├── http.go
    │   └── view.go
    └── web #web view
        ├── http.go
        └── view.go
```

# 入口
``` go
func Main() {
    NewController().Run(config)
}
```
``` go
func NewController() *Controller {
    ctl := &Controller{
        Logger:  log.NewPrefixLogger("controller"),
        updates: util.NewBroadcast(),
        cmds:    make(chan command),
        views:   make([]mvc.View, 0),
        state:   make(chan mvc.State),
    }

    return ctl
}
```
执行启动逻辑
``` go
func (ctl *Controller) Run(config *Configuration) {
    // Save the configuration
    ctl.config = config

    var model *ClientModel

    // 创建model（核心逻辑所在）
    if ctl.model == nil {
        // 不存在就创建
        model = ctl.SetupModel(config)
    } else {
        model = ctl.model.(*ClientModel)
    }

    // init the model
    var state mvc.State = model

    // 初始化web view
    var webView *web.WebView
    if config.InspectAddr != "disabled" {
        webView = web.NewWebView(ctl, config.InspectAddr)
        ctl.AddView(webView)
    }

    // 初始化term view
    var termView *term.TermView
    if config.LogTo != "stdout" {
        termView = term.NewTermView(ctl)
        ctl.AddView(termView)
    }

    // 将view关联到controller
    for _, protocol := range model.GetProtocols() {
        switch p := protocol.(type) {
        case *proto.Http:
            if termView != nil {
                ctl.AddView(termView.NewHttpView(p))
            }

            if webView != nil {
                ctl.AddView(webView.NewHttpView(p))
            }
        default:
        }
    }

    // 核心逻辑入口
    ctl.Go(ctl.model.Run)

    done := make(chan int)
    for {
        select {
        case obj := <-ctl.cmds:
            // 关闭command
            switch cmd := obj.(type) {
            case cmdQuit:
                msg := cmd.message
                go func() {
                    // 等待退出
                    ctl.doShutdown()
                    fmt.Println(msg)
                    done <- 1
                }()

            // 回放命令
            case cmdPlayRequest:
                ctl.Go(func() { ctl.model.PlayRequest(cmd.tunnel, cmd.payload) })
            }

        // 更新state
        case obj := <-updates:
            state = obj.(mvc.State)

        case ctl.state <- state:
        // 退出
        case <-done:
            return
        }
    }
}
```

最终进入一个死循环，似乎作者并没有打算让它和谐地退出
``` go
func (c *ClientModel) Run() {
    // how long we should wait before we reconnect
    maxWait := 30 * time.Second
    wait := 1 * time.Second

    for {
        // 开始发起连接请求，响应报文等
        // 注意是阻塞的，若该函数返回了，说明掉线了
        c.control()

        // 失败后，第一次状态未变，此时重置wait的时间间隔
        if c.connStatus == mvc.ConnOnline {
            wait = 1 * time.Second
        }

        // sleep，避免无畏的不停请求，浪费服务器资源
        time.Sleep(wait)
        // 失败了继续加大重试的间隔
        wait = 2 * wait
        wait = time.Duration(math.Min(float64(wait), float64(maxWait)))
        // 设置状态
        c.connStatus = mvc.ConnReconnecting
        // 刷新各种view
        c.update()
    }
}
```

## 退出
``` go
func (ctl *Controller) doShutdown() {
    ctl.Info("Shutting down")

    var wg sync.WaitGroup

    // wait for all of the views, plus the model
    wg.Add(len(ctl.views) + 1)

    // 关闭所有的view
    for _, v := range ctl.views {
        vClosure := v
        ctl.Go(func() {
            vClosure.Shutdown()
            wg.Done()
        })
    }

    // 关闭model（核心逻辑）
    ctl.Go(func() {
        ctl.model.Shutdown()
        wg.Done()
    })

    // 用WaitGroup等待多个goruntune
    wg.Wait()
}
```

# 主流程
``` go
func (c *ClientModel) control() {
    // establish control channel
    var (
        ctlConn conn.Conn
        err     error
    )

    // 向服务器发起连接
    if c.proxyUrl == "" {
        // simple non-proxied case, just connect to the server
        ctlConn, err = conn.Dial(c.serverAddr, "ctl", c.tlsConfig)
    } else {
        ctlConn, err = conn.DialHttpProxy(c.proxyUrl, c.serverAddr, "ctl", c.tlsConfig)
    }

    // authenticate with the server
    auth := &msg.Auth{
        ClientId:  c.id,
        OS:        runtime.GOOS,
        Arch:      runtime.GOARCH,
        Version:   version.Proto,
        MmVersion: version.MajorMinor(),
        User:      c.authToken,
    }

    // 发送连接报文
    if err = msg.WriteMsg(ctlConn, auth); err != nil {
        panic(err)
    }

    // 等待响应
    var authResp msg.AuthResp
    if err = msg.ReadMsgInto(ctlConn, &authResp); err != nil {
        panic(err)
    }

    if authResp.Error != "" {
        emsg := fmt.Sprintf("Failed to authenticate to server: %s", authResp.Error)
        c.ctl.Shutdown(emsg)
        return
    }

    c.id = authResp.ClientId
    c.serverVersion = authResp.MmVersion
    c.Info("Authenticated with server, client id: %v", c.id)
    // 更新view
    c.update()
    // 更新配置
    if err = SaveAuthToken(c.configPath, c.authToken); err != nil {
        c.Error("Failed to save auth token: %v", err)
    }

    // request tunnels
    reqIdToTunnelConfig := make(map[string]*TunnelConfiguration)
    for _, config := range c.tunnelConfig {
        // create the protocol list to ask for
        var protocols []string
        for proto, _ := range config.Protocols {
            protocols = append(protocols, proto)
        }

        // 注册tunnel
        reqTunnel := &msg.ReqTunnel{
            ReqId:      util.RandId(8),
            Protocol:   strings.Join(protocols, "+"),
            Hostname:   config.Hostname,
            Subdomain:  config.Subdomain,
            HttpAuth:   config.HttpAuth,
            RemotePort: config.RemotePort,
        }

        // 发送请求
        if err = msg.WriteMsg(ctlConn, reqTunnel); err != nil {
            panic(err)
        }

        // 存储这些配置，后面新建tunnel需要
        reqIdToTunnelConfig[reqTunnel.ReqId] = config
    }

    // 开始心跳
    lastPong := time.Now().UnixNano()
    c.ctl.Go(func() { c.heartbeat(&lastPong, ctlConn) })

    // main control loop
    for {
        var rawMsg msg.Message
        if rawMsg, err = msg.ReadMsg(ctlConn); err != nil {
            panic(err)
        }

        switch m := rawMsg.(type) {
        case *msg.ReqProxy:
            // 收到proxy请求，就向服务器发起一个proxy请求
            c.ctl.Go(c.proxy)

        case *msg.Pong:
            // 更新心跳
            atomic.StoreInt64(&lastPong, time.Now().UnixNano())

        case *msg.NewTunnel:
            // 注册tunnel确认
            if m.Error != "" {
                emsg := fmt.Sprintf("Server failed to allocate tunnel: %s", m.Error)
                c.Error(emsg)
                c.ctl.Shutdown(emsg)
                continue
            }

            tunnel := mvc.Tunnel{
                PublicUrl: m.Url,
                LocalAddr: reqIdToTunnelConfig[m.ReqId].Protocols[m.Protocol],
                Protocol:  c.protoMap[m.Protocol],
            }

            // 保存tunnel对象，用于view的数据呈现
            // 另外是申请proxy时，作校验
            c.tunnels[tunnel.PublicUrl] = tunnel
            // 更新状态
            c.connStatus = mvc.ConnOnline
            // 同步到view
            c.update()

        default:
            ctlConn.Warn("Ignoring unknown control message %v ", m)
        }
    }
}
```

## 心跳
``` go
func (c *ClientModel) heartbeat(lastPongAddr *int64, conn conn.Conn) {
    lastPing := time.Unix(atomic.LoadInt64(lastPongAddr)-1, 0)
    ping := time.NewTicker(pingInterval)
    pongCheck := time.NewTicker(time.Second)

    for {
        select {
        // 检查是否过期
        case <-pongCheck.C:
            lastPong := time.Unix(0, atomic.LoadInt64(lastPongAddr))
            needPong := lastPong.Sub(lastPing) < 0
            pongLatency := time.Since(lastPing)

            if needPong && pongLatency > maxPongLatency {
                c.Info("Last ping: %v, Last pong: %v", lastPing, lastPong)
                c.Info("Connection stale, haven't gotten PongMsg in %d seconds", int(pongLatency.Seconds()))
                return
            }

        // 定期发送心跳
        case <-ping.C:
            err := msg.WriteMsg(conn, &msg.Ping{})
            if err != nil {
                conn.Debug("Got error %v when writing PingMsg", err)
                return
            }
            lastPing = time.Now()
        }
    }
}
```

## proxy
当收到一个proxy请求时，会新开一个goruntune来处理
``` go
func (c *ClientModel) proxy() {
    var (
        remoteConn conn.Conn
        err        error
    )

    // 尝试连接服务器
    if c.proxyUrl == "" {
        remoteConn, err = conn.Dial(c.serverAddr, "pxy", c.tlsConfig)
    } else {
        remoteConn, err = conn.DialHttpProxy(c.proxyUrl, c.serverAddr, "pxy", c.tlsConfig)
    }
    defer remoteConn.Close()

    // 发送响应报文
    err = msg.WriteMsg(remoteConn, &msg.RegProxy{ClientId: c.id})
    if err != nil {
        remoteConn.Error("Failed to write RegProxy: %v", err)
        return
    }

    // 等待具体的业务报文，
    // 之所以需要这么个报文，是因为需要这个报文中的内容来定位tunnel，进而获取本地端的参数
    // 因为客户端也是一个数据中转，它一样有下游节点
    var startPxy msg.StartProxy
    if err = msg.ReadMsgInto(remoteConn, &startPxy); err != nil {
        remoteConn.Error("Server failed to write StartProxy: %v", err)
        return
    }

    // 找到tunnel
    tunnel, ok := c.tunnels[startPxy.Url]
    if !ok {
        remoteConn.Error("Couldn't find tunnel for proxy: %s", startPxy.Url)
        return
    }

    // 连接本地端
    start := time.Now()
    localConn, err := conn.Dial(tunnel.LocalAddr, "prv", nil)
    if err != nil {
        remoteConn.Warn("Failed to open private leg %s: %v", tunnel.LocalAddr, err)

        if tunnel.Protocol.GetName() == "http" {
            // ...
        }
        return
    }
    defer localConn.Close()

    m := c.metrics
    m.proxySetupTimer.Update(time.Since(start))
    m.connMeter.Mark(1)
    // 更新view
    c.update()
    m.connTimer.Time(func() {
        localConn := tunnel.Protocol.WrapConn(localConn,  \
            mvc.ConnectionContext{Tunnel: tunnel, ClientAddr: startPxy.ClientAddr})
        // 数据交换
        bytesIn, bytesOut := conn.Join(localConn, remoteConn)
        m.bytesIn.Update(bytesIn)
        m.bytesOut.Update(bytesOut)
        m.bytesInCount.Inc(bytesIn)
        m.bytesOutCount.Inc(bytesOut)
    })
    c.update()
}
```
一个典型的proxy开始报文如下
``` json
{"Type":"StartProxy","Payload":{"Url":"http://qjw.ngrok.com:10080","ClientAddr":"127.0.0.1:52630"}}
```

# 广播
ngrok可以同时支持若干view，为了实时同步状态等数据，
``` go
type Controller struct {
    // the model sends updates through this broadcast channel
    updates *util.Broadcast
}
```

``` go
func NewBroadcast() *Broadcast {
    b := &Broadcast{
        listeners: make([]chan interface{}, 0),
        reg:       make(chan (chan interface{})),
        unreg:     make(chan (chan interface{})),
        in:        make(chan interface{}),
    }

    go func() {
        for {
            select {
            // 取消注册
            case l := <-b.unreg:
                // remove L from b.listeners
                // this operation is slow: O(n) but not used frequently
                // unlike iterating over listeners
                oldListeners := b.listeners
                b.listeners = make([]chan interface{}, 0, len(oldListeners))
                // 这个删除操作很蛋疼
                for _, oldL := range oldListeners {
                    if l != oldL {
                        b.listeners = append(b.listeners, oldL)
                    }
                }
            // 注册操作
            case l := <-b.reg:
                b.listeners = append(b.listeners, l)
            // 刷新操作
            case item := <-b.in:
                for _, l := range b.listeners {
                    l <- item
                }
            }
        }
    }()

    return b
}

// 对外的刷新channel
func (b *Broadcast) In() chan interface{} {
    return b.in
}

// 生成一个注册channel，用于注册
func (b *Broadcast) Reg() chan interface{} {
    listener := make(chan interface{})
    b.reg <- listener
    return listener
}

// 取消注册一个channel
func (b *Broadcast) UnReg(listener chan interface{}) {
    b.unreg <- listener
}
```

## 使用
``` go
func (ctl *Controller) Update(state mvc.State) {
    ctl.updates.In() <- state
}
```
``` go
func NewWebView(ctl mvc.Controller, addr string) *WebView {
    // handle web socket connections
    http.HandleFunc("/_ws", func(w http.ResponseWriter, r *http.Request) {
        // 注册一个channel
        msgs := wv.wsMessages.Reg()
        // 退出是取消注册
        defer wv.wsMessages.UnReg(msgs)
        // 监听这个channel
        for m := range msgs {
            err := conn.WriteMessage(websocket.TextMessage, m.([]byte))
            if err != nil {
                // connection is closed
                break
            }
        }
    })
    return wv
}
```

# 拦截Http Request/Response
为了支持view呈现，需要拦截获取经过客户端的所有req/resp

## Protocol
每种tunnel 会有一种protocol,代码定义在src/github.com/qjw/ngrok/src/ngrok/proto/

在初始化view时，会设置proto到view对象中
``` go
for _, protocol := range model.GetProtocols() {
    switch p := protocol.(type) {
    case *proto.Http:
        if termView != nil {
            ctl.AddView(termView.NewHttpView(p))
        }

        if webView != nil {
            ctl.AddView(webView.NewHttpView(p))
        }
    default:
    }
}
```
``` go
func (v *TermView) NewHttpView(p *proto.Http) *HttpView {
    return newTermHttpView(v.ctl, v, p, 0, 12)
}
func (wv *WebView) NewHttpView(proto *proto.Http) *WebHttpView {
    return newWebHttpView(wv.ctl, wv, proto)
}
```

## 刷新
以webview为例
``` go
func newWebHttpView(ctl mvc.Controller, wv *WebView, proto *proto.Http) *WebHttpView {
    whv := &WebHttpView{
        Logger:       log.NewPrefixLogger("view", "web", "http"),
        webview:      wv,
        ctl:          ctl,
        httpProto:    proto,
        idToTxn:      make(map[string]*SerializedTxn),
        HttpRequests: util.NewRing(20),
    }
    // 实时刷新
    ctl.Go(whv.updateHttp)
    whv.register()
    return whv
}
```

``` go
func (whv *WebHttpView) updateHttp() {
    // open channels for incoming http state changes
    // and broadcasts
    txnUpdates := whv.httpProto.Txns.Reg()
    // 监听whv.httpProto.Txns
    for txn := range txnUpdates {
        // 获得实际的对象
        htxn := txn.(*proto.HttpTxn)

        // 。。。 刷新操作
}
```
``` go
type Http struct {
    // 可以看到Txns是一个广播对象
    Txns     *util.Broadcast
    reqGauge metrics.Gauge
    reqMeter metrics.Meter
    reqTimer metrics.Timer
}
```

在前面的代码中，会通过包装本地的conn
``` go
localConn := tunnel.Protocol.WrapConn(localConn, param)
```
``` go
func (h *Http) WrapConn(c conn.Conn, ctx interface{}) conn.Conn {
    tee := conn.NewTee(c)
    // 用一个管道将获取Request和Response串起来
    lastTxn := make(chan *HttpTxn)
    // 读取Request
    go h.readRequests(tee, lastTxn, ctx)
    // 读取Response
    go h.readResponses(tee, lastTxn)
    return tee
}
```
``` go
func (h *Http) readRequests(tee *conn.Tee, lastTxn chan *HttpTxn, connCtx interface{}) {
    defer close(lastTxn)

    for {
        // 不停地从tee的写tee中解析Request
        req, err := http.ReadRequest(tee.WriteBuffer())
        if err != nil {
            // no more requests to be read, we're done
            break
        }

        // golang's ReadRequest/DumpRequestOut is broken. Fix up the request so it works later
        req.URL.Scheme = "http"
        req.URL.Host = req.Host

        // 生成一个HttpTxn对象
        txn := &HttpTxn{Start: time.Now(), ConnUserCtx: connCtx}
        txn.Req = &HttpRequest{Request: req}
        if req.Body != nil {
            txn.Req.BodyBytes, txn.Req.Body, err = extractBody(req.Body)
            if err != nil {
                tee.Warn("Failed to extract request body: %v", err)
            }
        }

        // 发送到Req/Resp共享的channel，通知resp逻辑解析Response
        lastTxn <- txn
        // 通知view刷新
        h.Txns.In() <- txn
    }
}
```

``` go
func (h *Http) readResponses(tee *conn.Tee, lastTxn chan *HttpTxn) {
    for txn := range lastTxn {
        // 当req解析完之后，会触发resp解析
        // 从tee的读tee中不停地解析Response
        resp, err := http.ReadResponse(tee.ReadBuffer(), txn.Req.Request)
        txn.Duration = time.Since(txn.Start)
        h.reqTimer.Update(txn.Duration)
        if err != nil {
            tee.Warn("Error reading response from server: %v", err)
            // no more responses to be read, we're done
            break
        }

        // 更新HttpTxn对象的Response
        txn.Resp = &HttpResponse{Response: resp}
        // apparently, Body can be nil in some cases
        if resp.Body != nil {
            txn.Resp.BodyBytes, txn.Resp.Body, err = extractBody(resp.Body)
            if err != nil {
                tee.Warn("Failed to extract response body: %v", err)
            }
        }

        // 通知view刷新
        h.Txns.In() <- txn
    }
}
```

## Conn.Tee
``` go
func NewTee(conn Conn) *Tee {
    c := &Tee{
        rd:   nil,
        wr:   nil,
        Conn: conn,
    }

    c.readPipe.rd, c.readPipe.wr = io.Pipe()
    c.writePipe.rd, c.writePipe.wr = io.Pipe()

    // 读取的时候，一并拷贝一份到c.readPipe.wr，那么c.readPipe.rd就可读
    // 参考ReadBuffer
    c.rd = io.TeeReader(c.Conn, c.readPipe.wr)
    // 当写入的时候，一并写入c.writePipe.wr，那么c.writePipe.rz就可读
    // 参考WriteBuffer
    c.wr = io.MultiWriter(c.Conn, c.writePipe.wr)
    return c
}

func (c *Tee) ReadBuffer() *bufio.Reader {
    return bufio.NewReader(c.readPipe.rd)
}

func (c *Tee) WriteBuffer() *bufio.Reader {
    return bufio.NewReader(c.writePipe.rd)
}
```
