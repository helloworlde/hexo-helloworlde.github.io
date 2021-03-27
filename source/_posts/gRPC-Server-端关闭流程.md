---
title: gRPC Server 端关闭流程
date: 2020-12-05 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC Server 端关闭流程 

## 关闭 Server 

关闭 Server 可以使用 `shutdown` 或者 `shutdownNow` 方法

#### shutdown 

```java
server.shutdown().awaitTermination(10, TimeUnit.SECONDS);
```

- io.grpc.internal.ServerImpl#shutdown 

开始顺序的关闭 Server，已经存在的请求会继续执行，新的请求会被拒绝

```java 
public ServerImpl shutdown() {
    boolean shutdownTransportServers;
    synchronized (lock) {
        if (shutdown) {
            return this;
        }
        shutdown = true;
        shutdownTransportServers = started;
        if (!shutdownTransportServers) {
            transportServersTerminated = true;
            // 检查是否终止
            checkForTermination();
        }
    }
    if (shutdownTransportServers) {
        // 遍历所有的 Server 并关闭
        for (InternalServer ts : transportServers) {
            ts.shutdown();
        }
    }
    return this;
}
```

关闭时，首先会检查 Server 是否已经关闭了，如果已经关闭了，则抛出异常；如果没有关闭，则会修改关闭状态，返huan连接池，通知其他的锁；
然后会遍历所有的 Server，调用其 `shutdown` 方法进行关闭

- io.grpc.netty.NettyServer#shutdown

关闭 `NettySerer`，添加关闭事件监听器，并等待关闭；在监听器中会释放资源，关闭协议协调器，关闭 Transport 等

```java
public void shutdown() {
    // 如果 channel 已经关闭了，则返回
    if (channel == null || !channel.isOpen()) {
        // Already closed.
        return;
    }
    // 添加监听器，用于在关闭时释放资源，关闭协议，关闭 Transport 等
    channel.close().addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            if (!future.isSuccess()) {
                log.log(Level.WARNING, "Error shutting down server", future.cause());
            }
            InternalInstrumented<SocketStats> stats = listenSocketStats;
            listenSocketStats = null;
            if (stats != null) {
                channelz.removeListenSocket(stats);
            }
            sharedResourceReferenceCounter.release();
            protocolNegotiator.close();
            synchronized (NettyServer.this) {
                // 关闭 Transport
                listener.serverShutdown();
            }
        }
    });
    try {
        // 关闭 channel
        channel.closeFuture().await();
    } catch (InterruptedException e) {
        log.log(Level.FINE, "Interrupted while shutting down", e);
        Thread.currentThread().interrupt();
    }
}
```

- io.grpc.internal.ServerImpl.ServerListenerImpl#serverShutdown

监听 Server 关闭事件，根据关闭的状态，选择调用 `Transport` 的 `shutdown` 或者 `shutdownNow` 关闭 `ServerTransport`

```java
public void serverShutdown() {
    ArrayList<ServerTransport> copiedTransports;
    Status shutdownNowStatusCopy;
    // 复制 Transport 和状态
    synchronized (lock) {
        activeTransportServers--;
        if (activeTransportServers != 0) {
            return;
        }

        // transports collection can be modified during shutdown(), even if we hold the lock, due
        // to reentrancy.
        copiedTransports = new ArrayList<>(transports);
        shutdownNowStatusCopy = shutdownNowStatus;
        serverShutdownCallbackInvoked = true;
    }

    // 遍历 Transport，如果没有关闭状态，则调用shutdown 关闭，如果有状态，则调用 shutdownNow 立即关闭
    for (ServerTransport transport : copiedTransports) {
        if (shutdownNowStatusCopy == null) {
            transport.shutdown();
        } else {
            transport.shutdownNow(shutdownNowStatusCopy);
        }
    }

    synchronized (lock) {
        transportServersTerminated = true;
        // 是否终止的通知
        checkForTermination();
    }
}
```


- io.grpc.netty.NettyServerTransport#shutdown

最终在 Transport 中调用了 Netty Channel 的关闭方法，进行关闭

```java
@Override
public void shutdown() {
    if (channel.isOpen()) {
        channel.close();
    }
}
```

#### shutdownNow 

立即关闭 Server，已经存在的请求和新的请求都会被拒绝；尽管是强制的，但是 Server 并不会瞬间关闭

```java
server.shutdownNow();
```

- io.grpc.internal.ServerImpl#shutdownNow

立即关闭时，会先调用 `shutdown` 方法执行正常的关闭流程，然后修改关闭状态；遍历所有的 `ServerTransport`，调用其 `shutdownNow` 方法进行关闭

```java
public ServerImpl shutdownNow() {
    // 调用 shutdown 关闭 Transport
    shutdown();
    Collection<ServerTransport> transportsCopy;
    Status nowStatus = Status.UNAVAILABLE.withDescription("Server shutdownNow invoked");
    boolean savedServerShutdownCallbackInvoked;
    synchronized (lock) {
        if (shutdownNowStatus != null) {
            return this;
        }
        shutdownNowStatus = nowStatus;
        transportsCopy = new ArrayList<>(transports);
        savedServerShutdownCallbackInvoked = serverShutdownCallbackInvoked;
    }
    // 遍历 Transport 调用 shutdownNow
    if (savedServerShutdownCallbackInvoked) {
        //
        for (ServerTransport transport : transportsCopy) {
            transport.shutdownNow(nowStatus);
        }
    }
    return this;
}
```


- io.grpc.netty.NettyServerTransport#shutdownNow

`ServerTransport` 的 `shutdownNow` 会提交一个强制关闭的指令，并清空 channel，执行关闭

```java
public void shutdownNow(Status reason) {
    if (channel.isOpen()) {
        channel.writeAndFlush(new ForcefulCloseCommand(reason));
    }
}
```

