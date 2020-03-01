---
title: Jaeger初探
date: 2020-03-01 00:00:00
tags:
 - golang
 - python
 - grpc
categories:
 - 运营运维
---

Jaeger 是Uber 开源的分布式追踪系统，兼容OpenTracing 标准, 官网<https://github.com/jaegertracing/jaeger>

运行Sample

``` bash
$ docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.17
```

可以打开<http://localhost:16686>访问`Jeager UI`

# [概念](https://github.com/opentracing/specification/blob/master/specification.md)

> 这些概念可以对照Jeager UI加深/帮助理解

Traces in OpenTracing are defined implicitly by their Spans. In particular, a Trace can be thought of as a directed acyclic graph (DAG) of Spans, where the edges between Spans are called References.

Each Span encapsulates the following state:

+ An operation name
+ A start timestamp
+ A finish timestamp
+ A set of zero or more key:value `Span Tags`. The keys must be strings. The values may be strings, bools, or numeric types.
+ A set of zero or more `Span Logs`, each of which is itself a key:value map paired with a timestamp. The keys must be strings, though the values may be of any type. Not all OpenTracing implementations must support every value type.
+ A `SpanContext` (see below)
+ `References` to zero or more causally-related Spans (via the SpanContext of those related Spans)

Each SpanContext encapsulates the following state:

+ Any OpenTracing-implementation-dependent state (for example, trace and span ids) needed to refer to a distinct Span across a process boundary
+ Baggage Items, which are just key:value pairs that cross process boundaries

```
Causal relationships between Spans in a single Trace


        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)
```
```
Temporal relationships between Spans in a single Trace


––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```

参考
+ <https://opentracing.io/docs/getting-started/>
+ <https://github.com/opentracing-contrib/opentracing-specification-zh>
+ <https://wu-sheng.gitbooks.io/opentracing-io/content/>
+ <https://github.com/yurishkuro/opentracing-tutorial/tree/master/go>
+ <https://pjw.io/articles/2018/05/08/opentracing-explanations/>

# Golang
## grpc sample

``` bash
$ protoc -I ./ helloworld.proto --go_out=plugins=grpc:./
```

+ <https://github.com/grpc/grpc-go/blob/master/examples/helloworld/helloworld/helloworld.proto>
+ <https://github.com/grpc/grpc-go/blob/master/examples/helloworld/greeter_client/main.go>
+ <https://github.com/grpc/grpc-go/blob/master/examples/helloworld/greeter_server/main.go>

## Jeager Export

参考 <https://github.com/census-ecosystem/opencensus-go-exporter-jaeger/blob/master/example/main.go>

``` golang
import (
	"log"
	"contrib.go.opencensus.io/exporter/jaeger"
	"go.opencensus.io/trace"
)

func initTracer(name, agent string, sampling float64) (*jaeger.Exporter, error) {

	trace.ApplyConfig(trace.Config{DefaultSampler: trace.AlwaysSample()})
	// Register the Jaeger exporter to be able to retrieve
	// the collected spans.
	exporter, err := jaeger.NewExporter(jaeger.Options{
		CollectorEndpoint: agent,
		Process: jaeger.Process{
			ServiceName: name,
		},
	})
	if err != nil {
		log.Fatal(err)
	}
	trace.RegisterExporter(exporter)
	return exporter, nil
}

func main() {
	// 初始化trace服务器
	exporter, _ := initTracer("gotest", "http://localhost:14268/api/traces", 1)
	defer exporter.Flush()
}
```

## trace
``` golang
func dosth(ctx context.Context) {
	// 一个子span
	_, span := trace.StartSpan(ctx, "baidu")
	span.AddAttributes(trace.StringAttribute("name", "doget2"))
	span.SetStatus(trace.Status{Code: trace.StatusCodeOK, Message: "no error"})
	span.Annotate([]trace.Attribute{
		trace.Int64Attribute("len", int64(123456)),
		trace.StringAttribute("name", "dosth"),
	}, "Annotate")

	defer span.End()
	http.Get("http://www.baidu.com")
}
```

## Http trace

在python做各种三方库注入比较容易, golang需要手动处理, 参考<https://awesomeopensource.com/project/census-instrumentation/opencensus-go#Getting%20Started>

+ [net/http](https://godoc.org/go.opencensus.io/plugin/ochttp)
+ [gRPC](https://godoc.org/go.opencensus.io/plugin/ocgrpc)
+ [database/sql](https://godoc.org/github.com/opencensus-integrations/ocsql)
+ [Go kit](https://godoc.org/github.com/go-kit/kit/tracing/opencensus)
+ [Groupcache](https://godoc.org/github.com/orijtech/groupcache)
+ [Caddy webserver](https://godoc.org/github.com/orijtech/caddy)
+ [MongoDB](https://godoc.org/github.com/orijtech/mongo-go-driver)
+ [Redis gomodule/redigo](https://godoc.org/github.com/orijtech/redigo)
+ [Redis goredis/redis](https://godoc.org/github.com/orijtech/redis)
+ [Memcache](https://godoc.org/github.com/orijtech/gomemcache)

### client

较为完善的例子可参考<https://cloud.google.com/solutions/using-distributed-tracing-to-observe-microservice-latency-with-opencensus-and-stackdriver-trace>

``` golang
import (
	"context"
	fmt "fmt"
	"io/ioutil"
	"log"
	"net/http"

	"go.opencensus.io/plugin/ochttp/propagation/tracecontext"
	"go.opencensus.io/trace"
)

func doget(ctx context.Context) {
	// 调用子http服务的span
	name := "httpget"
	_, span := trace.StartSpan(ctx, name)
	url := "http://127.0.0.1:12345/sub"
	span.AddAttributes(trace.StringAttribute("url", url))
	defer span.End()

	// 调用Http服务
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		log.Fatalf("%v", err)
	}
	req = req.WithContext(ctx)

	format := &tracecontext.HTTPFormat{}
	format.SpanContextToRequest(span.SpanContext(), req)

	client := http.DefaultClient
	resp, err := client.Do(req)
	if err != nil {
		log.Fatalf("%v", err)
	}

	defer resp.Body.Close()
	s, err := ioutil.ReadAll(resp.Body)
	fmt.Printf(string(s))
}
```

### Server
``` golang
func hander(ctx context.Context, name string) func(http.ResponseWriter, *http.Request) {
	// 子服务router
	return func(w http.ResponseWriter, r *http.Request) {
		httpFormat := &tracecontext.HTTPFormat{}
		sc, ok := httpFormat.SpanContextFromRequest(r)
		var span *trace.Span = nil
		if ok {
			// 从header里面取spanContext, 做好衔接
			ctx, span = trace.StartSpanWithRemoteParent(ctx, name, sc)
		} else {
			ctx, span = trace.StartSpan(ctx, name)
		}

		// 增加Tag
		span.AddAttributes(trace.StringAttribute("name", fmt.Sprintf("name: %s", name)))
		span.SetStatus(trace.Status{Code: trace.StatusCodeOK, Message: "no error"})
		defer span.End()

		fmt.Fprintf(w, "Hello there!\n")
	}
}
```

## grpc trace

一个完整的例子可以参考<https://opencensus-website-snapshot.firebaseapp.com/gogrpc/>

### Client
``` golang
import (
	"context"
	"fmt"
	"log"

	"go.opencensus.io/plugin/ocgrpc"
	"go.opencensus.io/stats/view"
	"go.opencensus.io/trace"
	grpc "google.golang.org/grpc"
)
func grpc_client(ctx context.Context, address, name string) {
	// 一个子span
	ctx, span := trace.StartSpan(ctx, name)
	span.AddAttributes(trace.StringAttribute("grpc", fmt.Sprintf("name: %s", name)))
	defer span.End()

	if err := view.Register(ocgrpc.DefaultClientViews...); err != nil {
		log.Fatalf("Failed to register ocgrpc client views: %v", err)
	}
	conn, err := grpc.Dial(
		address,
		grpc.WithStatsHandler(&ocgrpc.ClientHandler{
			StartOptions: trace.StartOptions{
				Sampler: trace.AlwaysSample(),
			},
		}),
		grpc.WithInsecure(),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

    // 调用实际的rpc接口
    // grpc 生成的代码接口
	c := NewGreeterClient(conn)
	r, err := c.SayHello(ctx, &HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}
```

### Server
``` golang
func grpc_server(addr string) {
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	if err := view.Register(ocgrpc.DefaultServerViews...); err != nil {
		log.Fatalf("Failed to register ocgrpc server views: %v", err)
	}

	s := grpc.NewServer(grpc.StatsHandler(&ocgrpc.ServerHandler{}))
    // grpc生成的代码
	RegisterGreeterServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```


# Flask
```
Flask==0.12.4
opencensus==0.7.6
opencensus-ext-flask==0.7.3
opencensus-ext-jaeger==0.7.1
opencensus-ext-sqlalchemy==0.1.2
opencensus-ext-grpc==0.7.1
opencensus.ext.requests==0.7.2
```

## 初始化
```
def init_opencensus_tracing(app):
    from opencensus.ext.jaeger.trace_exporter import JaegerExporter
    from opencensus.ext.flask.flask_middleware import FlaskMiddleware
    from opencensus.trace import config_integration, samplers

    # 配置注入
    INTEGRATIONS = ['sqlalchemy', 'requests']
    exporter = JaegerExporter(
        service_name='flask_sample',
        agent_host_name="127.0.0.1",
        agent_port=6831,
    )
    sampler = samplers.ProbabilitySampler(rate=1)
    FlaskMiddleware(
        app,
        exporter=exporter,
        sampler=sampler,
        blacklist_paths=['health'],
    )
    config_integration.trace_integrations(INTEGRATIONS)
```

当使用`sqlalchemy`/`requests`做数据库操作/Http(s)操作时, 会自动生成新的`span`

## Python Grpc客户端
grpc不支持`配置集成(config integration)`

参考<https://github.com/census-instrumentation/opencensus-python/issues/677>

> This seems to be a doc issue, the grpc integration is done by explicitly using the interceptors (refer to the examples) instead of through config_integration.

``` bash
$ python -m grpc_tools.protoc --python_out ./ \
    --grpc_python_out ./ -I. ./helloworld.proto
```

``` python
    from opencensus.ext.grpc import client_interceptor
    import grpc
    from .helloworld_pb2 import HelloRequest
    from .helloworld_pb2_grpc import GreeterStub

    options = [("grpc.lb_policy_name", "round_robin")]

    global qrcoder_client
    qchannel = grpc.insecure_channel("127.0.0.1:12346", options)
    # 注入trace
    trace_interceptor = client_interceptor.OpenCensusClientInterceptor(
        host_port="127.0.0.1:12346"
    )
    qchannel = grpc.intercept_channel(qchannel, trace_interceptor)
    qrcoder_client = GreeterStub(qchannel)
    qrcoder_client.SayHello(HelloRequest(name="world"))
```

# 其他Export

上面一直都使用的[Jaeger](https://www.jaegertracing.io/), 其他包括参见<https://opencensus.io/exporters/supported-exporters/>


