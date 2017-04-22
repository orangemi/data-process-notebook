理解grpc in nodejs
=================

编写动力：grpc 在 nodejs 中的API文档写的非常的不友好，虽然知道如何起手，但不知道其原理。查看源码理解其些许原理和API

其中有一步使用了`grpc.Server.addProtoService(service: ProtoBuf.Reflect.Service, implements: Object<String, Function>)`这个API，这个API做了什么事情呢？
API把一个ProtoBuf的service的定义和实现绑定在了一块儿，例如protoBuf中定义了如下的简单service：
```protobuf
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
```
那么`addProtoService`就会找到这个定义，发现这个 service 属于简单一问一答的形式 `Unary RPC`，就把把内部的rpc方法注册为`unary`的方法。

grpc中有4种rpc方法，在官方文档中有介绍：http://www.grpc.io/docs/guides/concepts.html#rpc-life-cycle
- Unary RPC
- Server streaming RPC
- Client streaming RPC
- Bidirectional streaming RPC

这样在implement接入的对象中根据service定义的一样，需要建立不同的实现：

```protobuf
// sample.protobuf
syntax = "proto3";

service MongoProxy {
  rpc FindOne (Req) returns (Rep) {}
  rpc Find (Req) returns (stream Rep) {}
  rpc BatchFind (stream Req) returns (stream Rep) {}
  rpc BatchCount (stream Req) returns (Rep) {}
}

message Req {
  string condition = 1;
}

message Rep {
  string result = 1;
}

```

```js
// Unary
function Unary (call, callback) {
  console.log(call.request)
  callback(null, {})
}

// Server streaming RPC
function ServerStreaming (call) {
  console.log(call.request)
  while (condition) {
    call.write({})
  }
  call.end()
}

// Client streaming RPC
function ClientStreaming (call, callback) {
  call.on('data', (data) => {})
  call.on('end', () => { callback(null, {}) })
}

// Bidirectional streaming RPC
function Bidirectional (call) {
  call.on('data', (data) => { call.write({}) })
  call.on('end', () => { call.end(null, {}) })
}
```

这里必须注意的是，当rpc 方法是 Bidirectional, ServerStreaming 的时候，必须注意call.end的调用时机，有时候在异步处理中会发生先调用了call.end,再调用call.write的，这样就会报错，需要确保call.write和call.end的调用顺序，最好的方法是创建 `stream.Readable`，而不要自己调用 write 和 end 方法，例如：
```js
function Bidirectional (call) {
  let someStream = createReadableStream()
  call.on('data', (data) => { })
  call.on('end', () => { })
  someStream.pipe(call)
}
```