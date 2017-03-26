Title: Tracing gRPC Services In Go
Date: 2017-03-26 16:08
Category: Programming
Tags: Go, gRPC, Tracing, Monitoring


Built-in tracing with x/net/trace
---------------------------------

The `grpc` package contains a variable named `EnableTracing` that is set to true by default. When tracing is enabled, 
all requests and responses are recorded internally by the grpc library using the trace functions provided by the 
`x/net/trace` package. This package registers two HTTP handlers in `http.DefaultServeMux` to serve trace information at 
`/debug/events` and `/debug/requests` paths by default. Alternatively, `trace.Render(...)` and `trace.RenderEvents(...)` 
can be used in your own HTTP handlers to produce the same output.


At the time of writing, the `trace` package does not have have a persistence mechanism and the rendered information is 
a "live" view of traces held in memory. This makes it unsuitable for after-the-fact analysis.


```go

import (
    "http"
    "google.golang.org/grpc"
    _ "golang.org/x/net/trace"
)

func init() {
    grpc.EnableTracing = true
}

func startGRPCServer() {
    // start the gRPC server
}

func startDebugServer() {
    // Start the default http server and explicitly bind it to listen on localhost for security purposes
    // Accessing http://localhost:6060/debug/events or http://localhost:6060/debug/requests will show the
    // currently gathered traces.
    http.ListenAndServer("localhost:6060", nil) 
}
```


Dapper style distributed tracing
---------------------------------

There are several distributed tracing systems modelled after Google Dapper such as Zipkin, Jaeger, Stackdriver Trace etc.
[Opentracing](https://opentracing.io) is a vendor-neutral tracing implementation that supports multiple collector backends. 
The Go implementation of the Opentracing API can be found at [https://github.com/opentracing/opentracing-go](https://github.com/opentracing/opentracing-go)


The most obvious way to add trace information to gRPC invocations is to make use of server and client interceptors. Indeed, 
this is how the [grpc-opentracing](https://github.com/grpc-ecosystem/grpc-opentracing) library works. The client interceptor 
extracts the tracing information from the call context and adds it to the remote call as gRPC metadata. The server interceptor 
then extracts the tracing information from gRPC metadata and adds it to the call context on the server side. Unfortunately, 
this method only works flawlessy for unary RPCs. If you have any streaming RPCs, the current `ServerStreamInterceptor` does not
permit modification of the stream context and your trace data will be inaccurate. One way to work around this is to wrap the 
server stream and the new context in a new `ServerStream` as follows.


```go
type ServerStreamWrapper struct {
	stream grpc.ServerStream
	ctx    context.Context
}

func WrapServerStream(stream grpc.ServerStream, ctx context.Context) *ServerStreamWrapper {
	return &ServerStreamWrapper{stream: stream, ctx: ctx}
}

func (ssw *ServerStreamWrapper) SetHeader(md metadata.MD) error {
	return ssw.stream.SetHeader(md)
}

func (ssw *ServerStreamWrapper) SendHeader(md metadata.MD) error {
	return ssw.stream.SendHeader(md)
}

func (ssw *ServerStreamWrapper) SetTrailer(md metadata.MD) {
	ssw.stream.SetTrailer(md)
}

func (ssw *ServerStreamWrapper) Context() context.Context {
	return ssw.ctx
}

func (ssw *ServerStreamWrapper) SendMsg(m interface{}) error {
	return ssw.stream.SendMsg(m)
}

func (ssw *ServerStreamWrapper) RecvMsg(m interface{}) error {
	return ssw.stream.RecvMsg(m)
}
```

While this allows us to work around the immutable context in server streams, there are a few problems with this approach:

- There must be a good reason for why the gRPC developers have made server stream context unmodifiable
- If the gRPC `ServerStream` interface changes in the future, our code will break
- Even if the context modification problem is solved, we still do not have a mechanism to detect when the stream ends

The full source code of my attempt to use interceptors for tracing can be found here: [https://gist.github.com/charithe/b5956620fa6bb0caf668f857d70a5fc4](https://gist.github.com/charithe/b5956620fa6bb0caf668f857d70a5fc4)


Given the library limitations and the fact that streams can be live for quite long periods of time, attempting to trace them
might be inadvisable. However, I wanted to see if it was possible to do so, and the unorthodox solution I stumbled on to was to use
the `stats.Handler` interface now available in the gRPC package. This interface conveniently provides hooks into all the
interesting events that happen during the execution of a gRPC invocation, and can be harnessed to add instrumentation for tracing 
purposes. My implementation of this idea can be found at [https://github.com/charithe/otgrpc](https://github.com/charithe/otgrpc).

One of the issues I came across while implementing the library is that for client-only streams, the end of the stream is not
detected unless an error occurs. I have submitted a [pull request to the gRPC project](https://github.com/grpc/grpc-go/pull/1140) to fix this issue, and if it gets merged-in,
full tracing of both unary and streaming RPCs will become posible.


```go
// Server side

th := otgrpc.NewTraceHandler(tracer, otgrpc.WithPayloadLogging())
server := grpc.NewServer(grpc.StatsHandler(th))
// register gRPC services and start the server


// Client side

th := otgrpc.NewTraceHandler(tracer, otgrpc.WithPayloadLogging())
conn, err := grpc.Dial(address, grpc.WithStatsHandler(th))
// create client from the connection
```

A race session on a bi-directional stream, captured using `opentracing.MockTracer` looks as follows:

```
traceId=61, spanId=66, parentId=64, sampled=true, name=/test.TestSvc/BidiStreamRPC
	[map[component:gRPC span.kind:server]]
	2017-03-26 17:47:25.735035236 +0100 BST {Key:event ValueKind:string ValueString:Header received: Remote addr=@, Local addr=/tmp/testsvc}
	2017-03-26 17:47:25.735063884 +0100 BST {Key:event ValueKind:string ValueString:RPC started}
	2017-03-26 17:47:25.735071278 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735071278 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping0" }
	2017-03-26 17:47:25.73508489 +0100 BST {Key:event ValueKind:string ValueString:Header sent: Remote addr=%!s(<nil>), Local addr=%!s(<nil>)}
	2017-03-26 17:47:25.73508987 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.73508987 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping0" }
	2017-03-26 17:47:25.735143788 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735143788 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping1" }
	2017-03-26 17:47:25.735152847 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735152847 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping1" }
	2017-03-26 17:47:25.735187024 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735187024 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping2" }
	2017-03-26 17:47:25.735194692 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735194692 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping2" }
	2017-03-26 17:47:25.735232775 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735232775 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping3" }
	2017-03-26 17:47:25.735245634 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735245634 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping3" }
	2017-03-26 17:47:25.735303342 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735303342 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping4" }
	2017-03-26 17:47:25.735320488 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735320488 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping4" }
	2017-03-26 17:47:25.735357374 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=9}
	2017-03-26 17:47:25.735357374 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"OK" }
	2017-03-26 17:47:25.735371044 +0100 BST {Key:event ValueKind:string ValueString:Trailer sent}
	2017-03-26 17:47:25.735375683 +0100 BST {Key:event ValueKind:string ValueString:RPC ended}


traceId=61, spanId=64, parentId=62, sampled=true, name=/test.TestSvc/BidiStreamRPC
	[map[component:gRPC span.kind:client]]
	2017-03-26 17:47:25.734969911 +0100 BST {Key:event ValueKind:string ValueString:RPC started}
	2017-03-26 17:47:25.73499256 +0100 BST {Key:event ValueKind:string ValueString:Header sent: Remote addr=/tmp/testsvc, Local addr=@}
	2017-03-26 17:47:25.735003708 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735003708 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping0" }
	2017-03-26 17:47:25.735095398 +0100 BST {Key:event ValueKind:string ValueString:Header received: Remote addr=%!s(<nil>), Local addr=%!s(<nil>)}
	2017-03-26 17:47:25.73511693 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.73511693 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping0" }
	2017-03-26 17:47:25.735134045 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735134045 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping1" }
	2017-03-26 17:47:25.735166036 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735166036 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping1" }
	2017-03-26 17:47:25.735176586 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735176586 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping2" }
	2017-03-26 17:47:25.735207855 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735207855 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping2" }
	2017-03-26 17:47:25.735220191 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735220191 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping3" }
	2017-03-26 17:47:25.735265907 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735265907 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping3" }
	2017-03-26 17:47:25.735283455 +0100 BST {Key:event ValueKind:string ValueString:Payload sent: Wire length=12}
	2017-03-26 17:47:25.735283455 +0100 BST {Key:payload ValueKind:ptr ValueString:request:"ping4" }
	2017-03-26 17:47:25.735331558 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=7}
	2017-03-26 17:47:25.735331558 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"ping4" }
	2017-03-26 17:47:25.735365117 +0100 BST {Key:event ValueKind:string ValueString:Payload received: Wire length=4}
	2017-03-26 17:47:25.735365117 +0100 BST {Key:payload ValueKind:ptr ValueString:response:"OK" }
	2017-03-26 17:47:25.735377619 +0100 BST {Key:event ValueKind:string ValueString:Trailer received}
	2017-03-26 17:47:25.735394863 +0100 BST {Key:event ValueKind:string ValueString:RPC ended}


traceId=61, spanId=62, parentId=0, sampled=true, name=grpc_op
```

