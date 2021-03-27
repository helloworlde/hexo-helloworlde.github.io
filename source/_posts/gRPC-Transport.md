---
title: gRPC Transport
date: 2020-10-22 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC Transport 

Transport 分为 `ClientTransport` 和 `ServerTransport`，分别用于客户端和服务端

![grpc-source-code-transport-classes-diagram.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-transport-classes-diagram.png)

## ClientTransport

`ClientTransport` 与真正的 IP 地址是一一对应的，用于建立连接，创建流

### 实现

`ClientTransport` 有多个继承接口和实现类:
- 支持失败的实现类 `FailingClientTransport`，客户端启动时创建的流会快速失败
- 支持生命周期管理的接口 `ManagedClientTransport`
	- 支持延迟处理的实现类 `DelayedClientTransport`
	- 基于连接的接口 `ConnectionClientTransport`
		- 基于 Netty 实现的 `NettyClientTransport`
		- 基于 OkHttp 实现的 `OkHttpClientTransport` 
		- 用于进程内处理请求 `InProcessTransport`
		- 支持代理的抽象实现类 `ForwardingConnectionClientTransport`
			- 支持授权检查的实现类 `CallCredentialsApplyingTransportFactory`
			- 用于统计的 `CallTracingTransport`

### 方法

- ClientTransport#newStream 

创建新的流，用于给远程服务端发送消息

```java
ClientStream newStream(MethodDescriptor<?, ?> method, Metadata headers, CallOptions callOptions);
```

- ClientTransport#ping 

ping 远程的端点，当收到 ack 之后，会使用所给的 Executor 执行回调

```java
void ping(PingCallback callback, Executor executor);
```

- ManagedClientTransport#start

启动 Transport，尝试建立连接，并启动监听器

```java
Runnable start(Listener listener);
```

### 监听器

Transport 的 Listener 用于监听 Transport 事件，进行相应的处理

- 监听就绪事件

```java
void transportReady();
```

- 监听使用中事件

```java
void transportInUse(boolean inUse);
```

- 监听关闭事件

```java
void transportShutdown(Status s);
```

- 监听终止事件

```java
void transportTerminated();
```

### ClientTransport 流程

Transport 在 Channel 退出空闲模式时会被创建，然后通过 `start` 方法启动，建立连接，当连接成功后触发 ready 回调，然后更新 LB 状态为 READY，然后可以执行发送请求到服务端

#### 1. 创建 Transport 

有两个地方可以创建 Transport：
 - LB 通过 `handleResolvedAddresses`处理地址时通过`ManagedChannelImpl.SubchannelImpl#requestConnection`建立连接，会创建 Transport
 - `ClientCallImpl#start` 时，通过 `ClientTransportProvider#get` 方法获取 Transport

这两个方法最终在 `InternalSubchannel#obtainActiveTransport` 中调用 `startNewTransport` 执行创建 

##### 启动 Transport

`io.grpc.internal.InternalSubchannel#startNewTransport`

```java
private void startNewTransport() {
    // 获取当前地址
    SocketAddress address = addressIndex.getCurrentAddress();
    // 创建 Transport，并使用 CallTracingTransport 封装
    ConnectionClientTransport transport = new CallTracingTransport(transportFactory.newClientTransport(address, options, transportLogger), callsTracer);

    // 创建 Transport 监听器，建立连接
    Runnable runnable = transport.start(new TransportListener(transport, address));
    if (runnable != null) {
        syncContext.executeLater(runnable);
    }
}
```

##### 执行创建 Transport 

`io.grpc.netty.NettyChannelBuilder.NettyTransportFactory#newClientTransport`

```java
public ConnectionClientTransport newClientTransport(SocketAddress serverAddress,
                                                    ClientTransportOptions options,
                                                    ChannelLogger channelLogger) {
  // 存活时间
  final AtomicBackoff.State keepAliveTimeNanosState = keepAliveTimeNanos.getState();
  Runnable tooManyPingsRunnable = new Runnable() {
    @Override
    public void run() {
      keepAliveTimeNanosState.backoff();
    }
  };

  // 构建 Netty 的 Transport
  NettyClientTransport transport = new NettyClientTransport(/*..*/);
  return transport;
}
```

####  2. 启动监听器，建立连接

`io.grpc.netty.NettyClientTransport#start`

```java
public Runnable start(Listener transportListener) {
    // 创建 ClientTransportLifecycleManager
    lifecycleManager = new ClientTransportLifecycleManager(Preconditions.checkNotNull(transportListener, "listener"));
    // 建立连接
    SocketAddress localAddress = localSocketPicker.createSocketAddress(remoteAddress, eagAttributes);
    if (localAddress != null) {
        channel.connect(remoteAddress, localAddress);
    } else {
        channel.connect(remoteAddress);
    }

    // 开始 keepAlive 监听
    if (keepAliveManager != null) {
        keepAliveManager.onTransportStarted();
    }

    return null;
}
```

##### 触发 ready 回调

当 Netty 的 Handler 第一次接收到返回的消息时触发回调

- `io.grpc.netty.NettyClientHandler.FrameListener#onSettingsRead`

```java
public void onSettingsRead(ChannelHandlerContext ctx, Http2Settings settings) {
  if (firstSettings) {
    firstSettings = false;
    lifecycleManager.notifyReady();
  }
}
```

- `io.grpc.netty.ClientTransportLifecycleManager#notifyReady`

```java
public void notifyReady() {
    // 如果已经是 Ready 或者处于 SHUTDOWN 状态，则返回
    if (transportReady || transportShutdown) {
        return;
    }
    transportReady = true;
    listener.transportReady();
}
```

- `io.grpc.internal.InternalSubchannel.TransportListener#transportReady`

```java
public void transportReady() {
    syncContext.execute(new Runnable() {
        @Override
        public void run() {
            reconnectPolicy = null;
            // 如果是 SHUTDOWN 则关闭 Transport
            if (shutdownReason != null) {
                transport.shutdown(shutdownReason);
            } else if (pendingTransport == transport) {
                activeTransport = transport;
                pendingTransport = null;
                // 如果是等待 READY，则更新状态为 READY
                gotoNonErrorState(READY);
            }
        }
    });
}
```

最终由 `LoadBalancer` 处理，将 `Subchannel` 的连接状态改为 `READY`，返回非空的 `PickResult`，这样就可以执行向 Server 端发送的请求