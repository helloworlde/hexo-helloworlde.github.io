---
title: gRPC Server 端请求处理流程
date: 2020-12-15 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC Server 端请求处理流程

[TOC]

## 初始化

1. 创建并启动 ServerTransport

在 Server 启动的时候，最终调用 `NettyServer` 的 `start()` 方法，为 `ServerBootstrap` 添加了 `ChannelInitializer`，最终，当有新的连接建立时，会由 `NettyServerHandler` 调用该类的 `initChannel` 方法，初始化一个 `NettyServerTransport`

-  io.grpc.netty.NettyServer#start

在初始化 Netty Channel 时，会先创建 `NettyServerTransport`，然后调用监听器的 `Transport` 创建事件，添加一个超时取消任务；
然后会调用 `Transport` 的 `start` 方法启动 `Transport`

```java
b.childHandler(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(Channel ch) {
        // 构建基于 Netty 的 ServerTransport
        NettyServerTransport transport = new NettyServerTransport(/*...*/);
        ServerTransportListener transportListener;

        synchronized (NettyServer.this) {
            // 调用监听器回调，Transport 创建事件
            transportListener = listener.transportCreated(transport);
        }

        // 启动监听器
        transport.start(transportListener);
        ChannelFutureListener loopReleaser = new LoopReleaser();
        channelDone.addListener(loopReleaser);
        ch.closeFuture().addListener(loopReleaser);
    }
});
```

- io.grpc.netty.NettyServerTransport#start

在启动 `Transport` 时，会为当前的 `Transport` 创建一个处理器，并绑定到 Netty  的 Channel 中

```java
public void start(ServerTransportListener listener) {
    this.listener = listener;

    // 为 pipeline 创建 Netty Handler
    grpcHandler = createHandler(listener, channelUnused);

    // 创建 Handler
    ChannelHandler negotiationHandler = protocolNegotiator.newHandler(grpcHandler);
    ChannelHandler bufferingHandler = new WriteBufferingAndExceptionHandler(negotiationHandler);

    // 添加监听器
    ChannelFutureListener terminationNotifier = new TerminationNotifier();
    channelUnused.addListener(terminationNotifier);
    channel.closeFuture().addListener(terminationNotifier);

    channel.pipeline().addLast(bufferingHandler);
}
```

## 处理请求

当 Server 与 Client 的连接建立成功之后，可以开始处理请求

#### 请求整体处理流程
1. 读取 `Settings` 帧，触发 `Transport` `ready` 事件
2. 读取 `Header` 帧，触发 `FrameListener#onHeadersRead` 事件
	3. 由 `NettyServerHandler` 处理
		4. 根据 `Header` 里面的信息，获取相应的方法
		4. 将 HTTP 流转换为 `NettyServerStream`
		5. 触发 `Transport#streamCreated` 事件
			6. 检查编解码、解压缩等信息，创建可取消的上下文
			11. 初始化流监听器
			6. 提交 `StreamCreated` 任务
		7. 触发 `NettyServerStream.TransportState#onStreamAllocated` 事件
			8. 提交 `OnReady` 任务
9. 执行 `StreamCreated` 任务
	10. 根据方法名查找方法定义
	11. 调用 `startCall` 开始处理
		12. 遍历拦截器，使用拦截器包装方法处理器
		13. 调用 `startWrappedCall` 处理
			14. 创建 `ServerCallImpl` 实例
			15. 通过方法定义的请求处理器 `startCall` 方法处理
				16. 创建响应观察器 `ServerCallStreamObserverImpl` 实例
				17. 调用 `call.request()` 获取指定数量的消息
					18. 提交 `RequestRunnable` 任务获取指定数量的消息
				18. 创建调用监听器 `UnaryServerCallListener`
			19. 创建 `ServerStreamListenerImpl` 流监听器实例
20. 执行 `OnReady` 任务
	21. 调用 `UnaryServerCallListener#onReady` 处理 `Ready` 事件
		22. 修改 `ready` 状态
		23. 如果有 `onReadyHandler` 任务，则执行
24. 执行 `RequestRunnable` 任务
	25. 要求指定数量的消息
	25. 修改等待投递的消息数量
	26. 调用 `deliver` 方法投递
		27. 如果有待投递的消息，根据类型进行投递
			28. 当消息类型是消息体时，处理消息体
				29. 读取消息体的流
				30. 调用 `MessageFramer.Listener#messagesAvailable` 事件，通知新的消息
				31. 提交 `MessagesAvailable` 任务
   32. 调用 `MessageDeframer#close` 方法关闭帧
	   33. 调用流监听器半关闭事件
	   34. 提交 `HalfClosed` 任务
31.  执行 `MessagesAvailable` 任务
	32. 从 `MessageProducer` 中获取消息，解析为请求对象
	33. 调用 `SeverCall.Listener#onMessage` 方法处理消息
		34. 将 `request` 对象赋值给相应的对象，该对象会在 `halfClose` 时处理
35. 执行 `HalfClosed` 任务
	36. 调用 `invoke` 方法，处理业务逻辑
		37. 根据方法 ID，使用相应的实现调用业务逻辑
			38. 调用 `StreamObserver#onNext` 发送响应
				39. 发送响应 `Header`
					40. 设置编码和压缩的请求头
					41. 写入 `Header`
				40. 发送响应 `body`
					41. 将响应对象序列化为流
					42. 写入响应
					43. 清空缓存
			44. 调用 `StreamObserver#onComplete` 完成请求
				45. 使用 `OK` 状态关闭调用
					46. 修改关闭状态
					47. 调用流关闭事件
						48. 关闭帧
						49. 将响应状态加入响应元数据中
						50. 修改 `TransportState` 的状态
						51. 写入响应元数据，发送给客户端
	37. 冻结响应
	38. 如果 `ready` 状态，再次执行 `onReady` 事件
	39. 当流关闭时，调用 `TransportState#complete` 事件
		40. 关闭监听器
		41. 提交 `Closed` 任务
	42. 执行 `Closed` 任务
		43. 调用 `stream#complete`  事件
		44. 取消上下文

#### 1. 读取 Settings 帧

- io.grpc.netty.NettyServerHandler.FrameListener#onSettingsRead

当读取到 Settings 帧时，会调用 onSettingsRead 方法，同时会同时 Transport 监听器 ready 事件

```java
public void onSettingsRead(ChannelHandlerContext ctx, Http2Settings settings) {
    if (firstSettings) {
        firstSettings = false;
        // 通知 Transport ready
        attributes = transportListener.transportReady(negotiationAttributes);
    }
}
```

- io.grpc.internal.ServerImpl.ServerTransportListenerImpl#transportReady

会通知 Transport Ready 事件，会遍历 `ServerTransportFilter` 通知，默认没有 `ServerTransportFilter` 的实现

```java
public Attributes transportReady(Attributes attributes) {
    // 如果有握手超时回调，则取消
    handshakeTimeoutFuture.cancel(false);
    handshakeTimeoutFuture = null;

    // 遍历 TransportFilter，通知 ready 事件并获取 attributes
    for (ServerTransportFilter filter : transportFilters) {
        attributes = Preconditions.checkNotNull(filter.transportReady(attributes), "Filter %s returned null", filter);
    }
    this.attributes = attributes;
    return attributes;
}
```

#### 2. 接收 header

当 Server 接收到 Client 发送的 header 后，经过 Netty 处理，最终调用 `onHeadersRead` 开始处理流

- io.grpc.netty.shaded.io.grpc.netty.NettyServerHandler.FrameListener#onHeadersRead

接收 header 帧

```java
public void onHeadersRead(ChannelHandlerContext ctx, int streamId, Http2Headers headers, int streamDependency, short weight, boolean exclusive, int padding, boolean endStream) throws Http2Exception {
    if (NettyServerHandler.this.keepAliveManager != null) {
        NettyServerHandler.this.keepAliveManager.onDataReceived();
    }

    NettyServerHandler.this.onHeadersRead(ctx, streamId, headers);
}
```

- io.grpc.netty.NettyServerHandler.FrameListener#onHeadersRead

开始处理 header

```java
public void onHeadersRead(ChannelHandlerContext ctx,
                          int streamId,
                          Http2Headers headers,
                          int streamDependency,
                          short weight,
                          boolean exclusive,
                          int padding,
                          boolean endStream) throws Http2Exception {
    if (keepAliveManager != null) {
        keepAliveManager.onDataReceived();
    }
    // 最终会创建流并出发流创建事件
    NettyServerHandler.this.onHeadersRead(ctx, streamId, headers);
}
```

- io.grpc.netty.NettyServerHandler#onHeadersRead

会根据请求的 Header 信息，查找服务和方法，校验请求类型，请求方法，传输编码等内容；然后根据 HTTP2 流 Id，获取对应的流，将其转换为 `NettyServerStream`；调用 `Transport` 的 `onStreamCreated` 事件
向线程池中提交 `StreamCreated`，然后调用 `onStreamAllocated` 方法通知流 `StreamListener`的`onReady`事件提交`OnReady`任务

```java
private void onHeadersRead(ChannelHandlerContext ctx, int streamId, Http2Headers headers) throws Http2Exception {
  try {
    // 删除斜杠获取方法限定名称
    CharSequence path = headers.path();

    // 方法限定名，即包含服务名和方法名
    String method = path.subSequence(1, path.length()).toString();
    // 获取 HTTP 流
    Http2Stream http2Stream = requireHttp2Stream(streamId);

    // 将 header 转为 metadata
    Metadata metadata = Utils.convertHeaders(headers);
    // 创建支持统计的上下文
    StatsTraceContext statsTraceCtx = StatsTraceContext.newServerContext(streamTracerFactories, method, metadata);

    // 创建流的声明
    NettyServerStream.TransportState state = new NettyServerStream.TransportState(
            this,
            ctx.channel().eventLoop(),
            http2Stream,
            maxMessageSize,
            statsTraceCtx,
            transportTracer,
            method);

    try {
      // 获取请求的 authority
      String authority = getOrUpdateAuthority((AsciiString) headers.authority());
      // 创建 Server 端的流
      NettyServerStream stream = new NettyServerStream(ctx.channel(),
              state,
              attributes,
              authority,
              statsTraceCtx,
              transportTracer);

      // 触发监听器，通知流创建事件，查找相应处理器，开始处理流，会提交 StreamCreated 任务到线程池中
      transportListener.streamCreated(stream, method, metadata);
      // 会提交 OnReady 任务到线程池中，通知 Stream Ready
      state.onStreamAllocated();
      http2Stream.setProperty(streamKey, state);
    }
  } catch (Exception e) {
    logger.log(Level.WARNING, "Exception in onHeadersRead()", e);
    throw newStreamException(streamId, e);
  }
}
```

#### 3. 流创建事件

```java
transportListener.streamCreated(stream, method, metadata);
```

- io.grpc.internal.ServerImpl.ServerTransportListenerImpl#streamCreated

检查并初始化流的编解码，解压缩等信息；创建可需取消的上下文，选择要执行的线程池，初始化流监听器，最终提交流创建任务

```java
private void streamCreatedInternal(final ServerStream stream,
                                   final String methodName,
                                   final Metadata headers,
                                   final Tag tag) {
    final Executor wrappedExecutor;

    if (executor == directExecutor()) {
        wrappedExecutor = new SerializeReentrantCallsDirectExecutor();
        stream.optimizeForDirectExecutor();
    } else {
        // 否则使用指定的 Executor 执行
        wrappedExecutor = new SerializingExecutor(executor);
    }

    // 创建可以取消的上下文
    final Context.CancellableContext context = createContext(headers, statsTraceCtx);

    // 流事件监听器，处理流的所有生命周期事件
    final JumpToApplicationThreadServerStreamListener jumpListener = new JumpToApplicationThreadServerStreamListener(wrappedExecutor, executor, stream, context, tag);
    stream.setListener(jumpListener);

    // 提交流创建任务
    wrappedExecutor.execute(new StreamCreated());
}
```

接下来会执行流 ready 的任务

#### 4. 流 ready 事件

```java
state.onStreamAllocated();
```

- io.grpc.internal.AbstractStream.TransportState#onStreamAllocated

流分配，会调用流 ready 事件

```java
protected void onStreamAllocated() {
    checkState(listener() != null);
    synchronized (onReadyLock) {
        checkState(!allocated, "Already allocated");
        allocated = true;
    }
    notifyIfReady();
}
```

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#onReady

最终会调用 onReady 提交流的 `OnReady` 任务

```java
public void onReady() {
    try {
        callExecutor.execute(new OnReady());
    }
}
```

#### 5. 执行流创建任务 

执行 `StreamCreated`任务

- io.grpc.internal.ServerImpl.ServerTransportListenerImpl#streamCreated

执行 `StreamCreated`任务时，会先根据方法名称从注册器中查找对应的方法处理器，然后调用 startCall 方法进行处理

```java
// 流创建任务处理
    final class StreamCreated extends ContextRunnable {
 
        private void runInternal() {
            ServerStreamListener listener = NOOP_LISTENER;
            try {
                // 根据方法名称获取方法定义
                ServerMethodDefinition<?, ?> method = registry.lookupMethod(methodName);
                // 如果没有则从回退的方法注册器中查找
                if (method == null) {
                    method = fallbackRegistry.lookupMethod(methodName, stream.getAuthority());
                }
                // 如果没有则方法不存在，返回 UNIMPLEMENTED，关闭流，取消上下文
                if (method == null) {
                    Status status = Status.UNIMPLEMENTED.withDescription("Method not found: " + methodName);
                    stream.close(status, new Metadata());
                    context.cancel(null);
                    return;
                }
                // 如果方法存在，则开始调用
                listener = startCall(stream, methodName, method, headers, context, statsTraceCtx, tag);
            } catch (Throwable t) {
                stream.close(Status.fromThrowable(t), new Metadata());
                context.cancel(null);
                throw t;
            } finally {
                 jumpListener.setListener(listener);
            }
           
            final class ServerStreamCancellationListener implements Context.CancellationListener {
                @Override
                public void cancelled(Context context) {
                    Status status = statusFromCancelled(context);
                    if (DEADLINE_EXCEEDED.getCode().equals(status.getCode())) {
                        stream.cancel(status);
                    }
                 }
            }
            context.addListener(new ServerStreamCancellationListener(), directExecutor());
         }
    }
```

- io.grpc.internal.ServerImpl.ServerTransportListenerImpl#startCall

会获取方法的处理器，然后遍历拦截器，封装处理器，调用 `startWrappedCall` 处理

```java
private <ReqT, RespT> ServerStreamListener startCall(ServerStream stream,
                                                     String fullMethodName,
                                                     ServerMethodDefinition<ReqT, RespT> methodDef,
                                                     Metadata headers,
                                                     Context.CancellableContext context,
                                                     StatsTraceContext statsTraceCtx,
                                                     Tag tag) {
    // 从方法描述获取调用处理器
    ServerCallHandler<ReqT, RespT> handler = methodDef.getServerCallHandler();
    // 遍历拦截器，为处理器添加拦截器
    for (ServerInterceptor interceptor : interceptors) {
        handler = InternalServerInterceptors.interceptCallHandler(interceptor, handler);
    }
    // 使用添加了拦截器后的处理器创建新的方法定义
    ServerMethodDefinition<ReqT, RespT> interceptedDef = methodDef.withServerCallHandler(handler);

    // 处理封装后的调用
    return startWrappedCall(fullMethodName, wMethodDef, stream, headers, context, tag);
}
```

- io.grpc.internal.ServerImpl.ServerTransportListenerImpl#startWrappedCall

创建请求处理器实例，然后调用方法处理器，开始处理请求，同时创建流监听器

```java
private <WReqT, WRespT> ServerStreamListener startWrappedCall(String fullMethodName,
                                                              ServerMethodDefinition<WReqT, WRespT> methodDef,
                                                              ServerStream stream,
                                                              Metadata headers,
                                                              Context.CancellableContext context,
                                                              Tag tag) {

    // 创建请求处理器
    ServerCallImpl<WReqT, WRespT> call = new ServerCallImpl<>(stream,
            methodDef.getMethodDescriptor(),
            headers,
            context,
            decompressorRegistry,
            compressorRegistry,
            serverCallTracer,
            tag);

    // 调用方法处理器，真正调用实现逻辑的方法
    ServerCall.Listener<WReqT> listener = methodDef.getServerCallHandler().startCall(call, headers);
    if (listener == null) {
        throw new NullPointerException("startCall() returned a null listener for method " + fullMethodName);
    }

    // 根据调用监听器创建新的流监听器
    return call.newServerStreamListener(listener);
}
```

- io.grpc.stub.ServerCalls.UnaryServerCallHandler#startCall

会创建响应观察器，要求指定数量的消息，并创建监听器

```java
public ServerCall.Listener<ReqT> startCall(ServerCall<ReqT, RespT> call, Metadata headers) {
    // 创建响应处理器
    ServerCallStreamObserverImpl<ReqT, RespT> responseObserver = new ServerCallStreamObserverImpl<>(call);
    // 会调用 io.grpc.internal.AbstractStream#request 方法获取消息
    call.request(2);
    // 返回监听器
    return new UnaryServerCallListener(responseObserver, call);
}
```

#### 6.  提交要求指定数量的消息任务

在执行 `StreamCreated` 任务时，会调用 `startCall` 方法，提交 `RequestRunnable`任务，要求指定数量的消息

- io.grpc.internal.AbstractStream#request

在执行 `StreamCreated` 任务时指定接收的帧的数量

```java
public final void request(int numMessages) {
    transportState().requestMessagesFromDeframer(numMessages);
}
```

- io.grpc.internal.AbstractStream.TransportState#requestMessagesFromDeframer

提交获取指定数量的帧的任务

```java
private void requestMessagesFromDeframer(final int numMessages) {
    // 如果不是线程安全的解帧器，则由 Transport 的线程执行
    class RequestRunnable implements Runnable {
        @Override
        public void run() {
            try {
                deframer.request(numMessages);
            } catch (Throwable t) {
                deframeFailed(t);
            }
        }
    }
    runOnTransportThread(new RequestRunnable());
}
```

#### 7. 执行流 ready 任务

执行 `OnReady` 任务

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#onReady

执行 onReady 时提交的 `OnReady` 任务

```java
    final class OnReady extends ContextRunnable {
        OnReady() {
            super(context);
        }

        @Override
        public void runInContext() {
            try {
                // 调用监听器的 ready 事件
                getListener().onReady();
            } catch (Throwable t) {
                internalClose(t);
                throw t;
            }
        }
    }
```

- io.grpc.stub.ServerCalls.UnaryServerCallHandler.UnaryServerCallListener#onReady

处理流 ready 事件，如果有 `onReadyHandler` 则会执行

```java
public void onReady() {
    // 将 ready 状态变为 true
    wasReady = true;
    // 如果响应有 readyHandler，则执行
    if (responseObserver.onReadyHandler != null) {
        responseObserver.onReadyHandler.run();
    }
}
```


#### 8. 执行读取指定数量的消息任务并提交有可用消息任务

执行 `RequestRunnable`任务

- RequestRunnable

```java
class RequestRunnable implements Runnable {
    @Override
    public void run() {
        try {
            deframer.request(numMessages);
        } catch (Throwable t) {
            deframeFailed(t);
        }
    }
}
```

- io.grpc.internal.MessageDeframer#request

读取指定数量的帧

```java
public void request(int numMessages) {
  if (isClosed()) {
    return;
  }
  pendingDeliveries += numMessages;
  deliver();
}
```

- io.grpc.internal.MessageDeframer#deliver

读取消息并投递给监听器

```java
private void deliver() {
  // 检查投递的状态
  if (inDelivery) {
    return;
  }
  inDelivery = true;
  try {
    // 如果没有停止投递，且有等待投递的消息，且读取成功，则根据相应状态进行处理
    while (!stopDelivery && pendingDeliveries > 0 && readRequiredBytes()) {
      switch (state) {
        case HEADER:
          // 处理 header
          processHeader();
          break;
        case BODY:
          // 处理body
          processBody();
          pendingDeliveries--;
          break;
        default:
          throw new AssertionError("Invalid state: " + state);
      }
    }

    // 如果已经停止投递，则关闭
    if (stopDelivery) {
      close();
      return;
    }

    if (closeWhenComplete && isStalled()) {
      close();
    }
  } finally {
    inDelivery = false;
  }
}
```

- io.grpc.internal.MessageDeframer#processBody

读取请求体并通知监听器有新的消息

```java
private void processBody() {
  // 读取请求体的流
  InputStream stream = compressedFlag ? getCompressedBody() : getUncompressedBody();
  nextFrame = null;
  // 通知监听器有新的消息
  listener.messagesAvailable(new SingleMessageProducer(stream));

  // 将状态改为处理 header
  state = State.HEADER;
  requiredLength = HEADER_LENGTH;
}
```

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#messagesAvailable

通知有新的消息可用，会提交 `MessageAvailable` 任务

```java
public void messagesAvailable(final MessageProducer producer) {
    try {
        // 执行任务
        callExecutor.execute(new MessagesAvailable());
    }
}
```

####  9. 执行有新的可用消息任务

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#messagesAvailable

通知有新的消息可用，会提交 `MessageAvailable` 任务

```java
    final class MessagesAvailable extends ContextRunnable {

        MessagesAvailable() {
            super(context);
        }

        @Override
        public void runInContext() {
            try {
                // 获取监听器，通知有新的消息
                getListener().messagesAvailable(producer);
            } catch (Throwable t) {
                internalClose(t);
                throw t;
            }
        }
    }
```

- io.grpc.internal.ServerCallImpl.ServerStreamListenerImpl#messagesAvailableInternal

执行时，会先将流解析为请求对象，然后调用监听器的 `onMessage`方法，处理消息 

```java
private void messagesAvailableInternal(final MessageProducer producer) {
    // 如果调用已经取消了，则关闭生产者
    if (call.cancelled) {
        GrpcUtil.closeQuietly(producer);
        return;
    }

    InputStream message;
    try {
        // 从生产者中获取消息，
        while ((message = producer.next()) != null) {
            try {
                // 将流解析为请求对象，发送给监听器
                listener.onMessage(call.method.parseRequest(message));
            } catch (Throwable t) {
                GrpcUtil.closeQuietly(message);
                throw t;
            }
            message.close();
        }
    } catch (Throwable t) {
        GrpcUtil.closeQuietly(producer);
        Throwables.throwIfUnchecked(t);
        throw new RuntimeException(t);
    }
}
```

- io.grpc.stub.ServerCalls.UnaryServerCallHandler.UnaryServerCallListener#onMessage

由监听器接收消息，并赋值给相应的对象，在 `halfClose` 事件时处理该请求

```java
public void onMessage(ReqT request) {
    // 如果已经接收到了一个请求，则返回错误
    if (this.request != null) {
        call.close(Status.INTERNAL.withDescription(TOO_MANY_REQUESTS), new Metadata());
        canInvoke = false;
        return;
    }

    // 延迟执行调用 method.invoke() 直到 onHalfClose() 以确保客户端执行了半关闭
    this.request = request;
}
```

#### 10. 提交半关闭请求任务

当执行完 `RequestRunnable` 任务完成时，会调用 `MessageDeframer#close` 方法关闭帧

- io.grpc.internal.MessageDeframer#close

```java
public void close() {
    if (isClosed()) {
      return;
    }
    boolean hasPartialMessage = nextFrame != null && nextFrame.readableBytes() > 0;
    try {
      if (fullStreamDecompressor != null) {
        hasPartialMessage = hasPartialMessage || fullStreamDecompressor.hasPartialData();
        fullStreamDecompressor.close();
      }
      if (unprocessed != null) {
        unprocessed.close();
      }
      if (nextFrame != null) {
        nextFrame.close();
      }
    } finally {
      fullStreamDecompressor = null;
      unprocessed = null;
      nextFrame = null;
    }
    listener.deframerClosed(hasPartialMessage);
  }
```

- io.grpc.internal.AbstractServerStream.TransportState#deframerClosed

会执行关闭帧，然后调用 Stream 的监听器，通知半关闭

```java
public void deframerClosed(boolean hasPartialMessage) {
    deframerClosed = true;
    // 是否到达流结尾
    if (endOfStream) {
        // 如果不需要立即关闭，且有未完成的消息，返回错误并抛出异常
        if (!immediateCloseRequested && hasPartialMessage) {
            deframeFailed(Status.INTERNAL.withDescription("Encountered end-of-stream mid-frame")
                                         .asRuntimeException());
            deframerClosedTask = null;
            return;
        }
        // 通知半关闭
        listener.halfClosed();
    }
    // 如果有解帧器关闭的任务，则执行
    if (deframerClosedTask != null) {
        deframerClosedTask.run();
        deframerClosedTask = null;
    }
}
```

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#halfClosed

提交半关闭任务

```java
public void halfClosed() {
    try {
        callExecutor.execute(new HalfClosed());
    } 
}
```

#### 11. 执行半关闭任务

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#halfClosed

执行半关闭任务

```java
    final class HalfClosed extends ContextRunnable {
        HalfClosed() {
            super(context);
        }

        @Override
        public void runInContext() {
            try {
                // 调用监听器的半关闭事件
                getListener().halfClosed();
            } catch (Throwable t) {
                internalClose(t);
                throw t;
            }
        }
    }
```

- io.grpc.stub.ServerCalls.UnaryServerCallHandler.UnaryServerCallListener#onHalfClose

最终在监听器中调用相应的方法处理器，处理请求，并冻结响应；还会再次调用 `onReady` 事件，如果有 `onReadyHandler` 会执行

```java
public void onHalfClose() {
    // 如果不能调用则直接返回
    if (!canInvoke) {
        return;
    }

    // 如果请求是 null，则返回错我
    if (request == null) {
        call.close(Status.INTERNAL.withDescription(MISSING_REQUEST), new Metadata());
        return;
    }

    // 执行方法调用
    method.invoke(request, responseObserver);
    // 处理了请求之后将请求置为 null
    request = null;
    // 冻结响应
    responseObserver.freeze();
    // 判断是否 ready
    if (wasReady) {
        // 因为在 halfClose 中调用，错过了来自 Transport 的 onReady 事件，从这里恢复
        // 即在 ready 之后用于执行 onReadyHandler
        onReady();
    }
}
```

- io.github.helloworlde.HelloServiceGrpc.MethodHandlers#invoke(Req, io.grpc.stub.StreamObserver<Resp>)

处理请求，这部分是生成的代码，会调用相应的实例，处理请求，并将响应内容通过 StreamObserver 发送出去

```java
public void invoke(Req request, io.grpc.stub.StreamObserver<Resp> responseObserver) {
    switch (methodId) {
        case METHODID_HOW_ARE_YOU:
            serviceImpl.howAreYou((io.github.helloworlde.HelloMessage) request,
                    (io.grpc.stub.StreamObserver<io.github.helloworlde.HelloResponse>) responseObserver);
            break;
        default:
            throw new AssertionError();
    }
}
```

## 处理响应

### 1. 执行业务逻辑处理

- io.github.helloworlde.service.HelloServiceImpl#howAreYou

需要实现生成的接口，在方法中实现逻辑，并将响应通过 `StreamObserver` 发送出去 

```java
public void howAreYou(HelloMessage request, StreamObserver<HelloResponse> responseObserver) {
    responseObserver.onNext(HelloResponse.newBuilder().setResult("Hello : " + request.getMessage()).build());
    responseObserver.onCompleted();
}
```

###  2. 发送响应内容

- io.grpc.stub.ServerCalls.ServerCallStreamObserverImpl#onNext

发送单个响应时，会先检查请求是否取消了，如果已经取消了，则会抛出错误；接着检查请求的状态，如果是已经丢弃或者完成，也会抛出异常

然后会检查是否发送了 header，如果没有发送，则会先发送 header；发送 header 完成后会发送消息

```java
public void onNext(RespT response) {
    // 如果已经被取消调用了，则判断是否有取消回调，如果没有则返回取消状态
    if (cancelled) {
        if (onCancelHandler == null) {
            throw Status.CANCELLED.withDescription("call already cancelled").asRuntimeException();
        }
        return;
    }
    // 检查是否已经丢弃或者完成
    checkState(!aborted, "Stream was terminated by error, no further calls are allowed");
    checkState(!completed, "Stream is already completed, no further calls are allowed");
    // 如果还没有发送 header，则发送 header
    if (!sentHeaders) {
        call.sendHeaders(new Metadata());
        // 将发送 header 设置为 true
        sentHeaders = true;
    }
    // 然后发送响应
    call.sendMessage(response);
}
```

#### 1. 发送响应 header

- io.grpc.internal.ServerCallImpl#sendHeadersInternal

设置 header 内容，发送 header

```java
private void sendHeadersInternal(Metadata headers) {
        // 丢弃编码的 key
        headers.discardAll(MESSAGE_ENCODING_KEY);
        // 设置压缩器类型
        headers.put(MESSAGE_ENCODING_KEY, compressor.getMessageEncoding());
        // 为流设置压缩器
        stream.setCompressor(compressor);

        // 丢弃消息编码的 key
        headers.discardAll(MESSAGE_ACCEPT_ENCODING_KEY);
        if (advertisedEncodings.length != 0) {
            headers.put(MESSAGE_ACCEPT_ENCODING_KEY, advertisedEncodings);
        }

        // 将调用 header 状态改为 true
        sendHeadersCalled = true;
        stream.writeHeaders(headers);
    }
```

- io.grpc.internal.AbstractServerStream#writeHeaders

将 header 内容写入帧中，会调用 Netty 相关的方法发送内容

```java
public final void writeHeaders(Metadata headers) {
    Preconditions.checkNotNull(headers, "headers");

    headersSent = true;
    abstractServerStreamSink().writeHeaders(headers);
}
```

#### 2. 发送响应内容

- io.grpc.internal.ServerCallImpl#sendMessageInternal

发送响应内容，会先检查是否已经发送了 header，且请求没有关闭，且响应的状态正确
如果都没有问题，则将响应内容序列化为流，然后发送并清空缓冲区

```java
private void sendMessageInternal(RespT message) {
    // 检查是否已经发送了 header，和调用是否已经被关闭
    checkState(sendHeadersCalled, "sendHeaders has not been called");
    checkState(!closeCalled, "call is closed");

    // 如果是 UNARY 或者 CLIENT_STREAMING 类型的消息，且已经发送过消息了，则不允许再发送，返回错误状态
    if (method.getType().serverSendsOneMessage() && messageSent) {
        internalClose(Status.INTERNAL.withDescription(TOO_MANY_RESPONSES));
        return;
    }

    // 将发送消息状态改为 true
    messageSent = true;
    try {
        // 将消息序列化为流，写入消息，清空流
        InputStream resp = method.streamResponse(message);
        stream.writeMessage(resp);
        stream.flush();
    } catch (RuntimeException e) {
        close(Status.fromThrowable(e), new Metadata());
    } catch (Error e) {
        close(Status.CANCELLED.withDescription("Server sendMessage() failed with Error"), new Metadata());
        throw e;
    }
}
```

- io.grpc.internal.AbstractStream#writeMessage

检查帧的状态，如果帧没有关闭，则将流的内容写入帧中，并关闭流
最终消息内容通过 Netty 的相关方法发送给客户端

```java
public final void writeMessage(InputStream message) {
    checkNotNull(message, "message");
    try {
        if (!framer().isClosed()) {
            // 写入消息体
            framer().writePayload(message);
        }
    } finally {
        GrpcUtil.closeQuietly(message);
    }
}
```

- io.grpc.internal.AbstractStream#flush

清空帧的缓冲，将所有内容都发送给客户端

```java
public final void flush() {
    // 如果帧还没有关闭，则清空帧
    if (!framer().isClosed()) {
        framer().flush();
    }
}
```

### 3. 完成请求

当调用 `responseObserver.onCompleted` 后，会开始处理请求完成的逻辑

- io.grpc.stub.ServerCalls.ServerCallStreamObserverImpl#onCompleted

会先检查请求的状态，如果已经被取消了，且没有取消处理任务，则直接抛出取消状态的异常
如果请求正常完成，会使用 OK 状态关闭情趣，修改请求状态为完成

```java
public void onCompleted() {
    // 如果已经被取消，则返回取消的状态
    if (cancelled) {
        if (onCancelHandler == null) {
            throw Status.CANCELLED.withDescription("call already cancelled").asRuntimeException();
        }
    } else {
        // 通知请求完成
        call.close(Status.OK, new Metadata());
        // 将完成状态改为 true
        completed = true;
    }
}
```

- io.grpc.internal.ServerCallImpl#closeInternal

使用指定的状态和响应元数据关闭请求
如果没有发送响应，则会取消请求，并返回 INTERNAL 状态的错误；如果请求正常完成，则调用流关闭的接口，完成请求

```java
private void closeInternal(Status status, Metadata trailers) {
    // 检查是否已经关闭
    checkState(!closeCalled, "call already closed");
    try {
        // 将关闭状态改为 true
        closeCalled = true;

        // 检查状态如果是 OK，且方法类型是 Server 端只能发送一次，且没有发送消息，则返回错误
        if (status.isOk() && method.getType().serverSendsOneMessage() && !messageSent) {
            internalClose(Status.INTERNAL.withDescription(MISSING_RESPONSE));
            return;
        }

        // 关闭流
        stream.close(status, trailers);
    } finally {
        // 统计结果
        serverCallTracer.reportCallEnded(status.isOk());
    }
}
```

- io.grpc.internal.AbstractServerStream#close

关闭流，将响应的 header 写入到帧中；最终通过 Netty 的方法将响应发送给客户端

```java
public final void close(Status status, Metadata trailers) {
    // 如果出站的流还未关闭，则将状态改为关闭
    if (!outboundClosed) {
        outboundClosed = true;
        // 从服务端关闭 framer
        endOfMessages();
        // 将响应状态加入到 header 中
        addStatusToTrailers(trailers, status);
        // 安全设置，无需同步，因为访问被严格控制，只有这里设置关闭状态，保证在这里之后读取
        // 给 Transport 设置响应状态
        transportState().setClosedStatus(status);
        // 将响应的 header 信息写入帧中
        abstractServerStreamSink().writeTrailers(trailers, headersSent, status);
    }
}
```

- io.grpc.internal.AbstractStream#endOfMessages

关闭帧

```java
protected final void endOfMessages() {
    framer().close();
}
```

####  1. 发送响应结尾 header

在关闭流时，会将相应的状态和其他 header 发送给客户端

- io.grpc.internal.AbstractServerStream#close

```java
abstractServerStreamSink().writeTrailers(trailers, headersSent, status);
```

- io.grpc.netty.NettyServerStream.Sink#writeTrailers

```java
public void writeTrailers(Metadata trailers, boolean headersSent, Status status) {
    try {
        Http2Headers http2Trailers = Utils.convertTrailers(trailers, headersSent);
        // 将发送 header 的指令写入到队列中
        writeQueue.enqueue(SendResponseHeadersCommand.createTrailers(transportState(), http2Trailers, status), true);
    }
}
```

#### 2. 提交关闭任务

当发送完响应 Header 和 body 时，会因为已经到达帧末尾，调用`closeStreamWhenDone` 方法进行关闭

- io.grpc.netty.shaded.io.grpc.netty.NettyServerHandler#closeStreamWhenDone

```java
private void closeStreamWhenDone(ChannelPromise promise, int streamId) throws Http2Exception {
    final TransportState stream = this.serverStream(this.requireHttp2Stream(streamId));
    promise.addListener(new ChannelFutureListener() {
        public void operationComplete(ChannelFuture future) {
            stream.complete();
        }
    });
}
```
 
 - io.grpc.internal.AbstractServerStream.TransportState#complete

然后调用流的完成事件，关闭监听器

```java
public void complete() {
    // 如果解帧器已经关闭了，则关闭监听器
    if (deframerClosed) {
        deframerClosedTask = null;
        closeListener(Status.OK);
    } else {
        // 如果还未关闭，则创建关闭监听器任务，并立即关闭解帧器
        deframerClosedTask = new Runnable() {
            @Override
            public void run() {
                closeListener(Status.OK);
            }
        };
        immediateCloseRequested = true;
        closeDeframer(true);
    }
}
```

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#closedInternal
提交流关闭任务`Closed`

```java
private void closedInternal(final Status status) {
    // 如果状态不是 OK，则直接提交关闭 Context 任务
    if (!status.isOk()) {
        cancelExecutor.execute(new ContextCloser(context, status.getCause()));
    }
    final class Closed extends ContextRunnable {
        Closed() {
            super(context);
        }

        @Override
        public void runInContext() {
            PerfMark.startTask("ServerCallListener(app).closed", tag);
            PerfMark.linkIn(link);
            try {
                // 调用监听器的关闭事件
                getListener().closed(status);
            } finally {
                PerfMark.stopTask("ServerCallListener(app).closed", tag);
            }
        }
    }

    callExecutor.execute(new Closed());
}
```

#### 3. 执行关闭任务

执行 `Closed` 任务

io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#closed$Closed

```java
final class Closed extends ContextRunnable {
    @Override
    public void runInContext() {
        try {
            // 调用监听器的关闭事件
            getListener().closed(status);
        } 
    }
}
```

- io.grpc.internal.ServerCallImpl.ServerStreamListenerImpl#closedInternal

根据状态通知流监听器完成或者取消，最终取消上下文

```java
private void closedInternal(Status status) {
    try {
        // 如果状态是 OK，通知监听器完成
        if (status.isOk()) {
            listener.onComplete();
        } else {
            // 否则将状态改为取消，通知监听器取消
            call.cancelled = true;
            listener.onCancel();
        }
    } finally {
        // 取消上下文
        context.cancel(null);
    }
}
```

- io.grpc.Context.CancellableContext#cancel
取消上下文，取消所有的超时时间任务

```java
public boolean cancel(Throwable cause) {
    boolean triggeredCancel = false;
    synchronized (this) {
        // 如果没有取消，则取消，并修改状态
        if (!cancelled) {
            cancelled = true;
            // 如果有等待取消的任务，则取消
            if (pendingDeadline != null) {
                pendingDeadline.cancel(false);
                pendingDeadline = null;
            }
            this.cancellationCause = cause;
            triggeredCancel = true;
        }
    }
    // 如果取消成功了，则通知监听器
    if (triggeredCancel) {
        notifyAndClearListeners();
    }
    return triggeredCancel;
}
```


