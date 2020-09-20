---
title: gRPC 重试流程
date: 2020-09-20 22:40:07
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 重试流程

当第一次调用失败，流监听器关闭的时候，会根据请求的处理状态和方法的配置，判断是否需要重试

请求的处理状态有三种，在`io.grpc.internal.ClientStreamListener.RpcProgress`中定义：
- `PROCESSED`: 请求被正常处理，按照返回的状态码决定是否要重试
- `REFUSED`: 没有被服务端的应用逻辑层处理，直接重试，不计入重试次数
- `DROPPED`:  请求被负载均衡丢弃了，不重试，如果是对冲则取消其他的对冲请求，直接提交


## 发起请求

- io.grpc.stub.ClientCalls#blockingUnaryCall

![grpc-java-blockingUnaryCall-diagram.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-java-blockingUnaryCall-diagram.png)

- io.grpc.internal.ClientCallImpl#startInternal

![grpc-java-client-call-start.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-java-client-call-start.png)

- io.grpc.internal.ManagedChannelImpl.ChannelTransportProvider#newRetriableStream

![grpc-java-transport-provider-new-retriable-stream.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-java-transport-provider-new-retriable-stream.png)

- io.grpc.internal.RetriableStream#start

![grpc-java-retriable-stream-start.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-java-retriable-stream-start.png)

- io.grpc.internal.RetriableStream#createSubstream

![grpc-java-retriable-stream-create-sub-stream.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-java-retriable-stream-create-sub-stream.png)

- io.grpc.internal.ManagedChannelImpl.ChannelTransportProvider#newRetriableStream#RetryStream#newSubstream

![grpc-java-retriable-stream-new-sub-stream.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-java-retriable-stream-new-sub-stream.png)



1. 通过生成的代码中的方法，调用 `io.grpc.stub.ClientCalls#blockingUnaryCall`

首先通过 channel 创建`ClientCall`，然后通过 `futureUnaryCall` 提交请求，返回 Future，根据返回的 Future 循环等待，通过`executor.waitAndDrain()`执行请求，直到 Future 完成，返回结果

```java
 public static <ReqT, RespT> RespT blockingUnaryCall(Channel channel,
                                                      MethodDescriptor<ReqT, RespT> method,
                                                      CallOptions callOptions,
                                                      ReqT req) {
    // 创建新的调用的 ClientCall，指定了调用类型和执行器
    ClientCall<ReqT, RespT> call = channel.newCall(method, callOptions.withOption(ClientCalls.STUB_TYPE_OPTION, StubType.BLOCKING)
                                                                      .withExecutor(executor));
    try {
      // 执行调用，发出请求
      ListenableFuture<RespT> responseFuture = futureUnaryCall(call, req);
      while (!responseFuture.isDone()) {
        try {
	      // 执行提交的 Runnable 
          executor.waitAndDrain();
        } catch (InterruptedException e) {
          interrupt = true;
          call.cancel("Thread interrupted", e);
        }
      }
      return getUnchecked(responseFuture);
    }
  }
```

2. 执行 unary 调用 `futureUnaryCall`

这里创建了 Future，通过 `asyncUnaryRequestCall` 继续调用

```java
  public static <ReqT, RespT> ListenableFuture<RespT> futureUnaryCall(ClientCall<ReqT, RespT> call, ReqT req) {
    // 初始化 GrpcFuture
    GrpcFuture<RespT> responseFuture = new GrpcFuture<>(call);
    // 将 GrpcFuture 包装为继承了 Listener 的 UnaryStreamToFuture，提交任务
    asyncUnaryRequestCall(call, req, new UnaryStreamToFuture<>(responseFuture));
    return responseFuture;
  }
```

3. 异步调用

执行调用，发送消息

```java
  private static <ReqT, RespT> void asyncUnaryRequestCall(ClientCall<ReqT, RespT> call,
                                                          ReqT req,
                                                          StartableListener<RespT> responseListener) {
    // 开始调用
    startCall(call, responseListener);
    try {
      // 发送消息
      call.sendMessage(req);
      // 半关闭连接
      call.halfClose();
    }
  }
```

4. 开始一次调用，通过 responseListener 处理返回响应

开始调用，并启动响应监听器

```java
  private static <ReqT, RespT> void startCall(ClientCall<ReqT, RespT> call,
                                              StartableListener<RespT> responseListener) {
    call.start(responseListener, new Metadata());
    responseListener.onStart();
  }
```

5. 执行请求调用

```java
  private void startInternal(final Listener<RespT> observer, Metadata headers) {
    checkState(stream == null, "Already started");
    checkState(!cancelCalled, "call was cancelled");
    checkNotNull(observer, "observer");
    checkNotNull(headers, "headers");

    // 如果已经取消了，则不创建流，通知监听器取消回调
    if (context.isCancelled()) {
      stream = NoopClientStream.INSTANCE;
      executeCloseObserverInContext(observer, statusFromCancelled(context));
      return;
    }

    // 压缩器
    final String compressorName = callOptions.getCompressor();
    Compressor compressor;
    if (compressorName != null) {
      compressor = compressorRegistry.lookupCompressor(compressorName);
      // 如果设置了压缩器名称，但是没有相应的压缩器，则返回错误，关闭监听器
      if (compressor == null) {
        stream = NoopClientStream.INSTANCE;
        Status status = Status.INTERNAL.withDescription(String.format("Unable to find compressor by name %s", compressorName));
        executeCloseObserverInContext(observer, status);
        return;
      }
    } else {
      compressor = Codec.Identity.NONE;
    }

    // 根据参数添加 Header
    prepareHeaders(headers, decompressorRegistry, compressor, fullStreamDecompression);

    // 最后期限
    Deadline effectiveDeadline = effectiveDeadline();
    boolean deadlineExceeded = effectiveDeadline != null && effectiveDeadline.isExpired();
    // 如果没有过期
    if (!deadlineExceeded) {
      // 如果打开了重试，则创建重试流
      if (retryEnabled) {
        stream = clientTransportProvider.newRetriableStream(method, callOptions, headers, context);
      } else {
        // 根据获取 ClientTransport
        ClientTransport transport = clientTransportProvider.get(new PickSubchannelArgsImpl(method, headers, callOptions));
        Context origContext = context.attach();
        try {
          // 创建流
          stream = transport.newStream(method, headers, callOptions);
        } finally {
          context.detach(origContext);
        }
      }
    } else {
      // 初始化超时失败的流
      stream = new FailingClientStream(DEADLINE_EXCEEDED.withDescription("ClientCall started after deadline exceeded: " + effectiveDeadline));
    }

    // 直接执行器
    if (callExecutorIsDirect) {
      stream.optimizeForDirectExecutor();
    }
    // 调用地址
    if (callOptions.getAuthority() != null) {
      stream.setAuthority(callOptions.getAuthority());
    }
    // 最大传入字节数
    if (callOptions.getMaxInboundMessageSize() != null) {
      stream.setMaxInboundMessageSize(callOptions.getMaxInboundMessageSize());
    }
    // 最大传出字节数
    if (callOptions.getMaxOutboundMessageSize() != null) {
      stream.setMaxOutboundMessageSize(callOptions.getMaxOutboundMessageSize());
    }
    // 最后执行时间
    if (effectiveDeadline != null) {
      stream.setDeadline(effectiveDeadline);
    }
    // 压缩器
    stream.setCompressor(compressor);
    // 解压缩流
    if (fullStreamDecompression) {
      stream.setFullStreamDecompression(fullStreamDecompression);
    }
    // 解压流注册器
    stream.setDecompressorRegistry(decompressorRegistry);
    // 记录开始调用
    channelCallsTracer.reportCallStarted();
    // 封装支持取消的流监听器
    cancellationListener = new ContextCancellationListener(observer);
    // 初始化支持 header 操作的 ClientStreamListener
    // 调用 start 方法，修改流的状态
    stream.start(new ClientStreamListenerImpl(observer));

    // 稍后将上下文取消传播到被调用端
    context.addListener(cancellationListener, directExecutor());

    if (effectiveDeadline != null
            // 如果上下文具有有效的截止日期，则无需安排额外的任务
            && !effectiveDeadline.equals(context.getDeadline())
            // 如果 channel 已终止，则无需安排额外的任务
            && deadlineCancellationExecutor != null
            // 如果截止时间已经到期，请让失败的流处理
            && !(stream instanceof FailingClientStream)) {
      // 提交任务，并返回回调
      deadlineCancellationNotifyApplicationFuture = startDeadlineNotifyApplicationTimer(effectiveDeadline, observer);
    }

    // 移除 context 和监听器
    if (cancelListenersShouldBeRemoved) {
      removeContextListenerAndCancelDeadlineFuture();
    }
  }
```


#### 计算下次请求时间间隔

下次重试请求的时间间隔不固定，由 `initialBackoffNanos`,`backoffMultiplier`, `maxBackoffNanos` 和随机数共同决定

- 第一次的延时

第一次延时是 初始延时 `initialBackoffNanos` 乘以随机数

在获得了第一次延时之后，会计算下一次延时；下一次延时是前一次延时 `nextBackoffIntervalNanos`乘以退避指数 `backoffMultiplier`，与最大延时 `maxBackoffNanos`比较，取最小的，然后再乘以随机数

```java
nextBackoffIntervalNanos = retryPolicy.initialBackoffNanos
backoffNanos = (long) (nextBackoffIntervalNanos * random.nextDouble())
```

- 下一次延时

在获得了第一次延时之后，会计算下一次延时；下一次延时是前一次延时 `nextBackoffIntervalNanos`乘以退避指数 `backoffMultiplier`，与最大延时 `maxBackoffNanos`比较，取最小的，然后再乘以随机数

```java
// 计算下一次延时
nextBackoffIntervalNanos = Math.min((long) (nextBackoffIntervalNanos * retryPolicy.backoffMultiplier), retryPolicy.maxBackoffNanos);

backoffNanos = (long) (nextBackoffIntervalNanos * random.nextDouble())
```