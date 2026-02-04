# gRPC-Java 交互方式与关键类

## 1) 四种交互方式通用流程（Mermaid，Netty 传输，含职责与 API 作用）

```mermaid
sequenceDiagram
  autonumber
  participant AppC as Client App<br>职责: 业务发起调用
  participant Stub as Client Stub<br>职责: 生成/发起 RPC 调用
  participant Call as ClientCall<br>职责: 管理单次 RPC 状态
  participant WQ as WriteQueue<br>职责: 写命令队列/切到 EventLoop
  participant NHC as Netty Client Handler<br>职责: 客户端帧编解码/事件分发
  participant NChC as Netty Client Channel<br>职责: 客户端 HTTP/2 读写通道
  participant Net as Network/TCP<br>职责: 传输链路
  participant NChS as Netty Server Channel<br>职责: 服务端 HTTP/2 读写通道
  participant NHS as Netty Server Handler<br>职责: 服务端帧编解码/事件分发
  participant SCall as ServerCall<br>职责: 服务端调用状态
  participant Svc as Service Impl<br>职责: 业务实现

  Note over AppC,Svc: 一元（Unary）
  AppC->>Stub: unaryCall(req) - 触发一元调用
  Stub->>Call: start(listener, headers) - 建立调用并注册回调
  Call->>WQ: enqueue CreateStreamCommand - 写请求头
  WQ->>NHC: createStream - writeHeaders
  NHC->>NChC: writeHeaders - 客户端出站
  NChC->>Net: TCP send HEADERS
  Net->>NChS: TCP recv HEADERS
  NChS->>NHS: channelRead - 服务端入站事件
  NHS->>SCall: onHeadersRead - 创建服务端调用
  Stub->>Call: sendMessage(req) - 发送请求消息
  Stub->>Call: halfClose() - 客户端发送完毕
  Stub->>Call: request(1) - 申请响应(一元/客户端流式内部会触发 call.request(2))
  Call->>WQ: enqueue SendGrpcFrameCommand - 写请求体
  WQ->>NHC: sendGrpcFrame - writeData
  NHC->>NChC: writeData - 客户端出站
  NChC->>Net: TCP send DATA
  Net->>NChS: TCP recv DATA
  NChS->>NHS: channelRead - 服务端入站事件
  NHS->>SCall: onMessage(req) - 分发到服务端调用
  SCall->>Svc: onMessage(req) - 进入业务处理
  Svc-->>SCall: sendMessage(resp) - 生成响应消息
  SCall-->>NHS: sendHeaders - 响应头
  NHS-->>NChS: writeHeaders - 服务端出站
  NChS-->>Net: TCP send HEADERS
  Net-->>NChC: TCP recv HEADERS
  NChC-->>NHC: channelRead - 客户端入站事件
  SCall-->>NHS: sendMessage(resp) - 响应体
  NHS-->>NChS: writeData - 服务端出站
  NChS-->>Net: TCP send DATA
  Net-->>NChC: TCP recv DATA
  NChC-->>NHC: channelRead - 客户端入站事件
  NHC-->>Call: onMessage(resp) - 客户端收到响应
  NHC-->>Call: onClose(status, trailers) - 结束调用

  Note over AppC,Svc: 服务端流式（Server Streaming）
  AppC->>Stub: serverStreamingCall(req) - 触发服务端流式
  Stub->>Call: start(listener, headers) - 建立调用并注册回调
  Call->>WQ: enqueue CreateStreamCommand - 写请求头
  WQ->>NHC: createStream - writeHeaders
  NHC->>NChC: writeHeaders - 客户端出站
  NChC->>Net: TCP send HEADERS
  Net->>NChS: TCP recv HEADERS
  NChS->>NHS: channelRead - 服务端入站事件
  NHS->>SCall: onHeadersRead - 创建服务端调用
  Stub->>Call: sendMessage(req) - 发送请求消息
  Stub->>Call: halfClose() - 客户端发送完毕
  Call->>WQ: enqueue SendGrpcFrameCommand - 写请求体
  WQ->>NHC: sendGrpcFrame - writeData
  NHC->>NChC: writeData - 客户端出站
  NChC->>Net: TCP send DATA
  Net->>NChS: TCP recv DATA
  NChS->>NHS: channelRead - 服务端入站事件
  NHS->>SCall: onMessage(req) - 分发到服务端调用
  SCall->>Svc: onMessage(req) - 进入业务处理
  SCall-->>NHS: sendHeaders - 响应头(首条响应前)
  NHS-->>NChS: writeHeaders - 服务端出站
  NChS-->>Net: TCP send HEADERS
  Net-->>NChC: TCP recv HEADERS
  NChC-->>NHC: channelRead - 客户端入站事件
  loop N responses
    Stub->>Call: request(1) - 拉取 1 条响应
    Svc-->>SCall: sendMessage(resp) - 生成响应消息
    SCall-->>NHS: sendMessage(resp) - 响应体
    NHS-->>NChS: writeData - 服务端出站
    NChS-->>Net: TCP send DATA
    Net-->>NChC: TCP recv DATA
    NChC-->>NHC: channelRead - 客户端入站事件
    NHC-->>Call: onMessage(resp) - 客户端收到响应
  end
  NHC-->>Call: onClose(status, trailers) - 结束调用

  Note over AppC,Svc: 客户端流式（Client Streaming）
  AppC->>Stub: clientStreamingCall() - 触发客户端流式
  Stub->>Call: start(listener, headers) - 建立调用并注册回调
  Call->>WQ: enqueue CreateStreamCommand - 写请求头
  WQ->>NHC: createStream - writeHeaders
  NHC->>NChC: writeHeaders - 客户端出站
  NChC->>Net: TCP send HEADERS
  Net->>NChS: TCP recv HEADERS
  NChS->>NHS: channelRead - 服务端入站事件
  NHS->>SCall: onHeadersRead - 创建服务端调用
  loop N requests
    Stub->>Call: sendMessage(req) - 发送请求消息
    Call->>WQ: enqueue SendGrpcFrameCommand - 写请求体
    WQ->>NHC: sendGrpcFrame - writeData
    NHC->>NChC: writeData - 客户端出站
    NChC->>Net: TCP send DATA
    Net->>NChS: TCP recv DATA
    NChS->>NHS: channelRead - 服务端入站事件
    NHS->>SCall: onMessage(req...) - 分发到服务端调用
    SCall->>Svc: onMessage(req...) - 进入业务处理
  end
  Stub->>Call: halfClose() - 客户端发送完毕
  Stub->>Call: request(1) - 申请响应(客户端流式为一元响应, 内部会触发 call.request(2))
  Svc-->>SCall: sendMessage(resp) - 生成响应消息
  SCall-->>NHS: sendHeaders - 响应头
  NHS-->>NChS: writeHeaders - 服务端出站
  NChS-->>Net: TCP send HEADERS
  Net-->>NChC: TCP recv HEADERS
  NChC-->>NHC: channelRead - 客户端入站事件
  SCall-->>NHS: sendMessage(resp) - 响应体
  NHS-->>NChS: writeData - 服务端出站
  NChS-->>Net: TCP send DATA
  Net-->>NChC: TCP recv DATA
  NChC-->>NHC: channelRead - 客户端入站事件
  NHC-->>Call: onMessage(resp) - 客户端收到响应
  NHC-->>Call: onClose(status, trailers) - 结束调用

  Note over AppC,Svc: 双向流式（Bidi Streaming）
  AppC->>Stub: bidiStreamingCall() - 触发双向流式
  Stub->>Call: start(listener, headers) - 建立调用并注册回调
  Call->>WQ: enqueue CreateStreamCommand - 写请求头
  WQ->>NHC: createStream - writeHeaders
  NHC->>NChC: writeHeaders - 客户端出站
  NChC->>Net: TCP send HEADERS
  Net->>NChS: TCP recv HEADERS
  NChS->>NHS: channelRead - 服务端入站事件
  NHS->>SCall: onHeadersRead - 创建服务端调用
  par client sends
    loop N requests
      Stub->>Call: sendMessage(req) - 发送请求消息
      Call->>WQ: enqueue SendGrpcFrameCommand - 写请求体
      WQ->>NHC: sendGrpcFrame - writeData
      NHC->>NChC: writeData - 客户端出站
      NChC->>Net: TCP send DATA
      Net->>NChS: TCP recv DATA
      NChS->>NHS: channelRead - 服务端入站事件
      NHS->>SCall: onMessage(req...) - 分发到服务端调用
      SCall->>Svc: onMessage(req...) - 进入业务处理
    end
    Stub->>Call: halfClose() - 客户端发送完毕
  and server sends
    SCall-->>NHS: sendHeaders - 响应头(首条响应前)
    NHS-->>NChS: writeHeaders - 服务端出站
    NChS-->>Net: TCP send HEADERS
    Net-->>NChC: TCP recv HEADERS
    NChC-->>NHC: channelRead - 客户端入站事件
    loop N responses
      Stub->>Call: request(1) - 拉取 1 条响应
      Svc-->>SCall: sendMessage(resp) - 生成响应消息
      SCall-->>NHS: sendMessage(resp) - 响应体
      NHS-->>NChS: writeData - 服务端出站
      NChS-->>Net: TCP send DATA
      Net-->>NChC: TCP recv DATA
      NChC-->>NHC: channelRead - 客户端入站事件
      NHC-->>Call: onMessage(resp) - 客户端收到响应
    end
  end
  NHC-->>Call: onClose(status, trailers) - 结束调用
```

## 2) 关键类关系图（Mermaid DSL，含职责与 API 作用）

```mermaid
classDiagram
  class ManagedChannelBuilder {
    <<职责: 创建通道配置>>
    +forAddress(host, port) 构建目标地址
    +usePlaintext() 关闭 TLS
    +build() 创建通道
  }
  class ManagedChannel {
    <<职责: 客户端通道>>
    +newCall(MethodDescriptor, CallOptions) 创建 ClientCall
    +shutdown() 关闭通道
  }
  class AbstractStub~T~ {
    <<职责: 生成具体 Stub>>
    +withDeadlineAfter() 设置超时
    +build(Channel, CallOptions) 构建新 Stub
  }
  class ClientCall~Req,Resp~ {
    <<职责: 单次 RPC 生命周期>>
    +start(Listener, Metadata) 注册回调并开始
    +request(int) 拉取响应
    +sendMessage(Req) 发送请求
    +halfClose() 发送完成
    +cancel(String, Throwable) 取消调用
  }
  class ClientCall_Listener~Resp~ {
    <<职责: 客户端回调>>
    +onMessage(Resp) 接收响应
    +onReady() 可继续发送
    +onClose(Status, Metadata) 完成/失败
  }
  class MethodDescriptor~Req,Resp~ {
    <<职责: 方法元数据>>
  }
  class CallOptions {
    <<职责: 调用配置>>
  }
  class ClientCalls {
    <<职责: Stub 到 ClientCall 适配>>
  }
  class StreamObserver~T~ {
    <<职责: 业务层流回调>>
    +onNext(T) 推送消息
    +onError(Throwable) 错误通知
    +onCompleted() 完成通知
  }

  class ServerBuilder {
    <<职责: 创建服务端>>
    +forPort(int) 设置端口
    +addService(BindableService) 注册服务
    +build() 构建实例
    +start() 启动服务
  }
  class Server {
    <<职责: 服务端生命周期>>
    +start() 启动
    +shutdown() 关闭
  }
  class BindableService {
    <<职责: 暴露服务定义>>
    +bindService() 生成 ServiceDefinition
  }
  class ServerServiceDefinition {
    <<职责: 方法到处理器映射>>
  }
  class ServerCall~Req,Resp~ {
    <<职责: 服务端单次调用>>
    +sendHeaders(Metadata) 发送响应头
    +sendMessage(Resp) 发送响应
    +request(int) 拉取请求
    +close(Status, Metadata) 结束调用
  }
  class ServerCall_Listener~Req~ {
    <<职责: 服务端回调>>
    +onMessage(Req) 接收请求
    +onHalfClose() 客户端发完
    +onCancel() 被取消
    +onComplete() 正常结束
    +onReady() 可继续发送
  }
  class ServerCalls {
    <<职责: Service 到 ServerCall 适配>>
  }

  class NettyClientChannel {
    <<职责: 客户端 HTTP/2 通道>>
  }
  class NettyClientHandler {
    <<职责: 客户端入站处理>>
  }
  class NettyServerChannel {
    <<职责: 服务端 HTTP/2 通道>>
  }
  class NettyServerHandler {
    <<职责: 服务端入站处理>>
  }

  ManagedChannelBuilder --> ManagedChannel : builds
  ManagedChannel --> ClientCall~Req,Resp~ : newCall()
  AbstractStub~T~ --> ManagedChannel : uses
  AbstractStub~T~ --> MethodDescriptor~Req,Resp~ : uses
  AbstractStub~T~ --> CallOptions : uses
  ClientCalls --> ClientCall~Req,Resp~ : creates
  ClientCall~Req,Resp~ --> ClientCall_Listener~Resp~ : callbacks
  ClientCalls --> StreamObserver~T~ : adapts

  ServerBuilder --> Server : builds
  BindableService --> ServerServiceDefinition : bindService()
  ServerBuilder --> ServerServiceDefinition : addService
  ServerCalls --> ServerCall~Req,Resp~ : uses
  ServerCall~Req,Resp~ --> ServerCall_Listener~Req~ : callbacks
  ServerCalls --> StreamObserver~T~ : adapts

  ClientCall~Req,Resp~ --> NettyClientChannel : writes
  NettyClientChannel --> NettyClientHandler : inbound
  NettyServerHandler --> ServerCall~Req,Resp~ : dispatch
  NettyServerChannel --> NettyServerHandler : inbound
```
