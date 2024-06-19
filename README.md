## GeeRPC

`GeeRPC`从零实现`go`语言的官方标准库`net/rpc`,并添加了协议交换、注册中心、服务发现、负载均衡、超时处理等特性

### 消息的编码方式
目前提供两种消息编码方式：`json` 和 `Gob`，将这两种方式的选择放入结构体`Option`中
客户端和服务端的通信报文分为`header` 和 `body` 两部分 ，客户端固定采用`json`编码，后续的 `header` 和 `body` 的编码方式由 `Option` 中的 `CodeType` 指定，服务端首先使用 `JSON` 解码 `Option`，然后通过 `Option` 的 `CodeType` 解码剩余的内容。
即报文将以这样的形式发送：
```

| Option{MagicNumber: xxx, CodecType: xxx} | Header{ServiceMethod ...} | Body interface{} |
| <------      固定 JSON 编码      ------>  | <-------   编码方式由 CodeType 决定   ------->|

```
在一次连接中，Option 固定在报文的最开始，Header 和 Body 可以有多个，即报文可能是这样的
```

| Option | Header1 | Body1 | Header2 | Body2 | ...

```

### 服务端
服务端实现了 `Accept` 方式，首先会循环等待 `socket` 连接建立，并开启子协程处理

子协程首先会反序列化得到 `Option` 实例，然后根据选择的消息类型进行消息的解码，随后 读取请求、处理请求、回复请求

- `handleRequest` 使用协程并发处理请求
- 回复请求是逐个回复的，否则容易造成回复报文交织

### 客户端

客户端支持异步和并发，实现接收响应和发送请求两个功能，并实现了两个暴露给用户的 `RPC` 服务调用接口，`Go` 是一个异步端口，`Call` 是一个同步端口

### 服务注册

`rpc` 框架本质是将结构体映射为服务，采用反射获取结构体的所有方法并且通过方法获取到该方法所有的参数类型和返回值

### 超时处理

在服务端和客户端分别增加超时处理机制，主要添加超时机制的点：

- 客户端创建连接时，在 `Option` 中设置超时时间
- 客户端`Client.Call()` 整个过程导致的超时（包含发送报文、等待处理、接收报文所有阶段）。使用context包，由用户自行决定是否创建具备超时检测能力的context对象
- 服务端处理报文，即 `Server.handleRequest` 超时。使用 `time.After()` 结合 `select + chan` 完成

