Title: Protocol Buffers and gRPC Tips for Go
Date: 2017-03-15 20:28
Category: Programming
Tags: Go, gRPC, Protobuf


This is a brain dump of tools and code snippets that come in handy when implementing gRPC services or using protobufs with Go.

## Libraries And Utilities

```shell
cat <<EOF | xargs go get -u
github.com/golang/protobuf/proto
github.com/golang/protobuf/protoc-gen-go
google.golang.org/grpc
github.com/gogo/protobuf/proto
github.com/gogo/protobuf/jsonpb
github.com/gogo/protobuf/protoc-gen-gogofast
github.com/gogo/protobuf/gogoproto
go.pedge.io/protoeasy/cmd/protoeasy
github.com/mwitkow/go-grpc-middleware
github.com/grpc-ecosystem/go-grpc-prometheus
EOF
```

## Gogoproto

[GogoProto](https://github.com/gogo/protobuf) is a fork of protobuf with additional goodies such as faster marshalers,
utility method generators and best of all, test and benchmark generators for the message definitions. 

```
syntax = "proto3";
package example;

import "gogoproto/gogo.proto";

option (gogoproto.marshaler_all) = true;
option (gogoproto.unmarshaler_all) = true;
option (gogoproto.sizer_all) = true;
option (gogoproto.populate_all) = true;
option (gogoproto.equal_all) = true;
option (gogoproto.testgen_all) = true;
option (gogoproto.benchgen_all) = true;
...
```

In addition to the above, there are [lots of other useful extensions](https://github.com/gogo/protobuf/blob/master/extensions.md) that can be added.

## Protoeasy

[Protoeasy](https://github.com/peter-edge/protoeasy-go) vastly simplifies the process of compiling protobufs and using 
protoc plugins such as `grpc`, `grpc-gateway` and `gogoproto`.

Following command generates protobuf, gRPC and gRPC gateway code with Gogoproto:

```
ROOT_PACKAGE=github.com/charithe/example
PROTO_PACKAGE=proto
protoeasy --gogo --go-import-path=$ROOT_PACKAGE --grpc --grpc-gateway $PROTO_PACKAGE/
```

## Tracing With Interceptors

This snippet uses Stackdrive Trace -- which lacks some features to effectively trace all requests. [Opentracing](http://opentracing.io/) 
and [Jaeger](https://uber.github.io/jaeger/) seem more promising and a [Stackdriver Trace reporter for Opentracing](https://github.com/lovoo/gcloud-opentracing) 
also exists. Unfortuantely, all of these libraries have some missing features/performance issues that need to be addressed. 

```go
func SpanFromMetadata(name string, ctx context.Context) (*trace.Span, bool) {
	if md, ok := metadata.FromContext(ctx); ok {
		traceContext, ok := md["x-cloud-trace-context"]
		if ok && len(traceContext) > 0 {
			s := trace.SpanFromHeader(name, traceContext[0])
			return s, true
		}
	}
    
	return traceClient.NewSpan(name), false
}

func GetUnaryServerInterceptor() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
		methodName := fmt.Sprintf("server:%s", info.FullMethod)
		span, ok := SpanFromMetadata(methodName, ctx)
		if !ok {
			span = trace.NewSpan(methodName)
		}
		newCtx := trace.NewContext(ctx, span)
		result, err := handler(newCtx, req)
		if err != nil {
			span.SetLabel("error", err.Error())
		}
		span.Finish()
		return result, err
	}
}

func GetStreamServerInterceptor() grpc.StreamServerInterceptor {
	return func(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
		methodName := fmt.Sprintf("server:%s", info.FullMethod)
		span, ok := SpanFromMetadata(methodName, ss.Context())
		if !ok {
			span = trace.NewSpan(methodName)
		}
		//TODO Stream context cannot be modified. We should add trace data through the SetHeader method
		err := handler(srv, ss)
		if err != nil {
			span.SetLabel("error", err.Error())
		}
		span.Finish()
		return err
	}
}

func GetUnaryClientInterceptor() grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		methodName := fmt.Sprintf("client:%s", method)
		span := trace.FromContext(ctx).NewChild(methodName)
		err := invoker(ctx, method, req, reply, cc, opts...)
		if err != nil {
			span.SetLabel("error", err.Error())
		}
		span.Finish()
		return err
	}
}

func GetStreamClientInterceptor() grpc.StreamClientInterceptor {
	return func(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
		methodName := fmt.Sprintf("client:%s", method)
		span := trace.FromContext(ctx).NewChild(methodName)
		cs, err := streamer(ctx, desc, cc, method, opts...)
		if err != nil {
			span.SetLabel("error", err.Error())
		}
		span.Finish()
		return cs, err
	}
}
```

## Chaining Interceptors

[go-grpc-middleware](https://github.com/mwitkow/go-grpc-middleware) allows chaining multiple interceptors -- which is quite handy.

```go
s := grpc.NewServer(
		grpc.StreamInterceptor(
			grpc_middleware.ChainStreamServer(
				GetStreamServerInterceptor(),
				grpc_prometheus.StreamServerInterceptor)),
		grpc.UnaryInterceptor(
			grpc_middleware.ChainUnaryServer(
				GetUnaryServerInterceptor(),
				grpc_prometheus.UnaryServerInterceptor)))
```

## Adding Interceptors To Google Cloud Libraries

```go
bigtable.NewClient(context.Background(),
		project,
		instance,
		option.WithGRPCConnectionPool(grpcConnPoolSize),
		option.WithGRPCDialOption(grpc.WithUnaryInterceptor(GetUnaryClientInterceptor())),
		option.WithGRPCDialOption(grpc.WithStreamInterceptor(GetStreamClientInterceptor())))

```

