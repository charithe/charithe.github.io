Title: gRPC pool for groupcache
Date: 2016-07-20 19:20
Category: Programming
Tags: Go, gRPC, Systems Programming

[Groupcache](https://github.com/golang/groupcache) is a system first written at Google to provide a caching layer for the 
[dl.google.com service](https://talks.golang.org/2013/oscon-dl.slide#1). It uses consistent hashing to 
distribute the key space over a group of machines. Each node is able to answer queries by first checking its' own cache 
or by forwarding the request to a peer that could have it cached. To avoid the thundering herd problem when there's a cache 
miss, only one request is allowed to read-through to the backend service. Any other requests for the same key would be blocked 
until the first request returns and the cache is populated.  

The default implementation of groupcache uses HTTP to communicate between peers. Whilst this is a perfectly good solution,
if you're attemmpting to incorporate groupcache into a [gRPC](http://grpc.io) based service, it might be more desirable to use
gRPC to communicate between the peers as well. [gcgrpcpool](https://github.com/charithe/gcgrpcpool) is a drop-in replacement 
for the default HTTPPool which implements peer communications using gRPC. 

```go
server := grpc.NewServer()

p := NewGRPCPool("127.0.0.1:5000", server)
p.Set(peerAddrs...)

getter := groupcache.GetterFunc(func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
    dest.SetString(...)
    return nil
})

groupcache.NewGroup("grpcPool", 1<<20, getter)
lis, err := net.Listen("tcp", "127.0.0.1:5000")
if err != nil {
    log.Fatalf("Failed to start server")
}

server.Serve(lis)
```

Because gRPC connections are long-lived, gcgrpcpool attempts to keep unchanged peer connections intact during a `Set` method call. 
A future improvement might be to use a custom gRPC `Balancer` implementation to maintain the connections. 

