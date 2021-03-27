---
title: gRPC Stream
date: 2020-11-08 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC Stream 

Stream 在 gRPC 中代表一个真正的请求，包含要发送的 消息；Stream 分为 `ClientStream` 和 `ServerStream`

![grpc-source-code-stream-diagram.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-stream-diagram.png)

## ClientStream 

`ClientStream` 接口继承 Stream 接口，有多个实现类或抽象实现类：
- `ForwardingClientStream`: 用于转发的 `ClientStream`，支持代理真正的流，可以用于触发一些动作，如统计等
- `NoopClientStream`: 不做任何操作的 `ClientStream`，用于空实现
	- `FailingClientStream`: 用于失败的 `ClientStream`，处理请求失败的场景
- `InProcessClientStream`: 进程内的 `ClientStream`，用于测试，不会发出实际请求
- `AbstractClientStream`: `ClientStream` 的抽象实现类，实现了部分基础操作，如发送header，写入消息，半关闭等
	- `NettyClientStream`: 基于 Netty 实现的 `ClientStream`，实现了基于 Netty 的帧操作等
	- `OkHttpClientStream`: 基于 OkHttp 实现的 `ClientStream`，实现了基于 OkHttp 的帧操作等

### 方法

- start 

开始一个新的流，对于每一个流，只能调用一次

```java
void start(ClientStreamListener listener);
```

- halfClose

从客户端关闭流，当关闭后，不能发送更多的消息，但是可以接收消息，只能调用一次

```java
void halfClose();
```

- cancel

异常终止流，当调用后不会再发送和接收消息，只有在 start 之后才可以被调用

```java
void cancel(Status reason);
```

- setDeadline

设置请求有效截止时间，过了这个时间之后就会终止请求执行

```java
void setDeadline(@Nonnull Deadline deadline);
```

- getAttributes

获取流的属性

```java
Attributes getAttributes();
```

### 监听器

ClientStreamListener 用于监听客户端流的事件

- onReady 

当接收得此事件说明 Transport 已经准备好发送附加消息了

```java
void onReady();
```

- messagesAvailable

当远程端点接收到消息时调用

```java
void messagesAvailable(MessageProducer producer);
```

- headersRead

当收到服务端返回的 header 时调用，如果没有 header 返回，则这个方法不会被调用

```java
void headersRead(Metadata headers);
```

- closed 

当流被关闭时调用

```java
void closed(Status status, Metadata trailers);

void closed(Status status, RpcProgress rpcProgress, Metadata trailers);
```

### ClientStream 流程

#### 发起请求

生成的代码中通过  `blockingUnaryCall` 开始发起请求

- io.grpc.stub.ClientCalls#blockingUnaryCall

```java
public static <ReqT, RespT> RespT blockingUnaryCall(Channel channel,
                                                    MethodDescriptor<ReqT, RespT> method,
                                                    CallOptions callOptions,
                                                    ReqT req) {
  // 构建任务队列和单线程的线程池
  ThreadlessExecutor executor = new ThreadlessExecutor();
  boolean interrupt = false;
  // 创建新的调用的 ClientCall，指定了调用类型和执行器
  ClientCall<ReqT, RespT> call = channel.newCall(method, callOptions.withOption(ClientCalls.STUB_TYPE_OPTION, StubType.BLOCKING)
                                                                    .withExecutor(executor));
  try {
    // 执行调用，发出请求
    ListenableFuture<RespT> responseFuture = futureUnaryCall(call, req);
    while (!responseFuture.isDone()) {
      try {
        executor.waitAndDrain();
      } catch (InterruptedException e) {
        interrupt = true;
        call.cancel("Thread interrupted", e);
      }
    }
    return getUnchecked(responseFuture);
  } catch (RuntimeException e) {
    throw cancelThrow(call, e);
  } catch (Error e) {
    throw cancelThrow(call, e);
  } finally {
    if (interrupt) {
      Thread.currentThread().interrupt();
    }
  }
}
```

创建 ClientCall 后调用 `futureUnaryCall` 开始发起请求，并返回用于获取结果的 `ListenableFuture`

```java
public static <ReqT, RespT> ListenableFuture<RespT> futureUnaryCall(ClientCall<ReqT, RespT> call, ReqT req) {
  // 初始化 GrpcFuture
  GrpcFuture<RespT> responseFuture = new GrpcFuture<>(call);
  // 将 GrpcFuture 包装为继承了 Listener 的 UnaryStreamToFuture，提交任务
  asyncUnaryRequestCall(call, req, new UnaryStreamToFuture<>(responseFuture));
  return responseFuture;
}
```

- io.grpc.stub.ClientCalls#asyncUnaryRequestCall

```java
private static <ReqT, RespT> void asyncUnaryRequestCall(ClientCall<ReqT, RespT> call,
                                                        ReqT req,
                                                        StartableListener<RespT> responseListener) {
  // 开始调用
  startCall(call, responseListener);
  try {
    // 发送消息，提交 BufferEntry 任务
    call.sendMessage(req);
    // 从客户端关闭流
    call.halfClose();
  } catch (RuntimeException e) {
    throw cancelThrow(call, e);
  } catch (Error e) {
    throw cancelThrow(call, e);
  }
}
```

#### 创建 ClientStream 

当调用了 `io.grpc.internal.ClientCallImpl#start` 方法后，会创建客户端流；
如果已经过了超时时间，则会使用 `DEADLINE_EXECEEDED` 状态创建 `FailingClientStream`；如果还为超时，则根据是否开启了重试创建相应的流

- io.grpc.internal.ClientCallImpl#startInternal

```java
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
    // 创建流
    stream = transport.newStream(method, headers, callOptions);
  }
} else {
  // 初始化超时失败的流
  stream = new FailingClientStream(DEADLINE_EXCEEDED.withDescription("ClientCall started after deadline exceeded: " + effectiveDeadline));
}
```

然后根据不同的实现，在 Transport 内创建流

- io.grpc.netty.NettyClientTransport#newStream

```java
public ClientStream newStream(MethodDescriptor<?, ?> method,
                              Metadata headers,
                              CallOptions callOptions) {

    // 如果 channel 是空的，则返回失败的 ClientStream
    if (channel == null) {
        return new FailingClientStream(statusExplainingWhyTheChannelIsNull);
    }

    StatsTraceContext statsTraceCtx = StatsTraceContext.newClientContext(callOptions, getAttributes(), headers);

    // 创建 NettyClientStream
    return new NettyClientStream(/**/);
}
```

#### 开始流

开始流，并指定监听器

- io.grpc.internal.AbstractClientStream#start

```java
public final void start(ClientStreamListener listener) {
    transportState().setListener(listener);
    // 如果不是 GET 请求，则发送 Header
    if (!useGet) {
        abstractClientStreamSink().writeHeaders(headers, null);
        headers = null;
    }
}
```

如果不是 GET 方法的请求，要先写入 Header

- io.grpc.netty.shaded.io.grpc.netty.NettyClientStream.Sink#writeHeaders

最终是通过创建 Netty 的指令，将 header 发送给服务端

```java
private void writeHeadersInternal(Metadata headers, byte[] requestPayload) {
    // 将 Header 转为 Netty 的 HTTP2 的 header
    // 根据方法名获取路径
    AsciiString defaultPath = (AsciiString) methodDescriptorAccessor.geRawMethodName(method);
    // 如果路径为 null，则设置路径为方法全名
    if (defaultPath == null) {
        defaultPath = new AsciiString("/" + method.getFullMethodName());
        methodDescriptorAccessor.setRawMethodName(method, defaultPath);
    }
    // 如果有 payload，则使用 GET 方法
    boolean get = (requestPayload != null);
    AsciiString httpMethod;
    // 如果是 GET 方法，则将负载加到请求路径中，并设置请求方法
    if (get) {
        // 将负载通过 base64 编码后添加到请求路径中
        defaultPath = new AsciiString(defaultPath + "?" + BaseEncoding.base64().encode(requestPayload));
        httpMethod = Utils.HTTP_GET_METHOD;
    } else {
        httpMethod = Utils.HTTP_METHOD;
    }
    // 将 Header 转为 Netty 的 HTTP2 header
    Http2Headers http2Headers = Utils.convertClientHeaders(headers, scheme, defaultPath, authority, httpMethod, userAgent);

    // 创建 ChannelFuture 的监听器
    ChannelFutureListener failureListener = new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            // 如果 Future 状态不是成功的
            if (!future.isSuccess()) {
                // 流创建失败时，如果流没有关闭，则关闭流；当 Channel 关闭时， Lifecycle Manager 更了解失败，
                // 尤其是在判断完成之前
                // 获取关闭状态
                Status s = transportState().handler.getLifecycleManager().getShutdownStatus();
                // 如果关闭状态是 null，则从失败的 Future 中获取失败状态
                if (s == null) {
                    s = transportState().statusFromFailedFuture(future);
                }
                // 上报 Transport 的状态
                transportState().transportReportStatus(s, true, new Metadata());
            }
        }
    };
    // Write the command requesting the creation of the stream.
    // 写入创建流的请求的指令，并添加失败的 Future 监听器
    writeQueue.enqueue(new CreateStreamCommand(http2Headers, transportState(), shouldBeCountedForInUse(), get),
            !method.getType().clientSendsOneMessage() || get)
              .addListener(failureListener);
}
```


#### 发起请求 

当在 `io.grpc.stub.ClientCalls#startCall`中调用了`responseListener.onStart()`后，会开始发送请求

- io.grpc.stub.ClientCalls.UnaryStreamToFuture#onStart

```java
void onStart() {
  responseFuture.call.request(2);
}
```  

- io.grpc.internal.ClientCallImpl#request

```java
public void request(int numMessages) {
    stream.request(numMessages);
}
```

- io.grpc.internal.AbstractStream#request

```java
public final void request(int numMessages) {
    transportState().requestMessagesFromDeframer(numMessages);
}
```

然后通过 Deframer 发送

- io.grpc.internal.AbstractStream.TransportState#requestMessagesFromDeframer

```java
private void requestMessagesFromDeframer(final int numMessages) {
    // 如果是线程安全的解帧器，则直接执行
    if (deframer instanceof ThreadOptimizedDeframer) {
        // 发送指定数量的消息
        deframer.request(numMessages);
        return;
    }
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

#### 发送消息

- io.grpc.internal.ClientCallImpl#sendMessageInternal

判断是否是可重试的流，如果是，则使用可重试的流发送消息，如果不是，则使用普通的流发送消息

```java
private void sendMessageInternal(ReqT message) {
  try {
    // 如果是重试流，则通过重试流的方法发送消息
    if (stream instanceof RetriableStream) {
      RetriableStream<ReqT> retriableStream = (RetriableStream<ReqT>) stream;
      retriableStream.sendMessage(message);
    } else {
      // 不是重试流，将消息转为流，发送
      stream.writeMessage(method.streamRequest(message));
    }
  } catch (RuntimeException e) {
    // 如果出错则取消请求
    stream.cancel(Status.CANCELLED.withCause(e).withDescription("Failed to stream message"));
    return;
  } catch (Error e) {
    stream.cancel(Status.CANCELLED.withDescription("Client sendMessage() failed with Error"));
    throw e;
  }
  // 对于 unary 请求，不用flush，因为接下来就是 halfClose, 这样就可以在消息最后搭载 END_STREAM=true，
  // 而无需打开损坏的流
  if (!unaryRequest) {
    stream.flush();
  }
}
```

- io.grpc.internal.AbstractStream#writeMessage

将消息内容转为流后，最终通过将消息传递给 Framer

```java
public final void writeMessage(InputStream message) {
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

 - io.grpc.internal.AbstractClientStream#deliverFrame

将 Framer 的内容传递给 Transport 

```java
public final void deliverFrame(WritableBuffer frame,
                               boolean endOfStream,
                               boolean flush,
                               int numMessages) {
    Preconditions.checkArgument(frame != null || endOfStream, "null frame before EOS");
    // 通过 netty 写入
    abstractClientStreamSink().writeFrame(frame, endOfStream, flush, numMessages);
}
```

- io.grpc.netty.NettyClientStream.Sink#writeFrameInternal

最终通过 Netty 的指令，将消息内容发送给服务端

```java
private void writeFrameInternal(WritableBuffer frame, boolean endOfStream, boolean flush, final int numMessages) {
    // 将 frame 转换为 ByteBuf
    ByteBuf bytebuf = frame == null ? EMPTY_BUFFER : ((NettyWritableBuffer) frame).bytebuf().touch();
    // 统计 ByteBuf 的可读字节数
    final int numBytes = bytebuf.readableBytes();
    // 如果字节数大于 0
    if (numBytes > 0) {
        // 将要出站的字节数添加到流控中
        onSendingBytes(numBytes);
        // 将发送 gRPC 帧命令添加到写队列中
        writeQueue.enqueue(new SendGrpcFrameCommand(transportState(), bytebuf, endOfStream), flush)
                  .addListener(new ChannelFutureListener() {
                      @Override
                      public void operationComplete(ChannelFuture future) throws Exception {
                          // 如果 future 成功，且 Transport 中的流不为 null
                          if (future.isSuccess() && transportState().http2Stream() != null) {
                              // 添加发送的字节数及统计
                              transportState().onSentBytes(numBytes);
                              NettyClientStream.this.getTransportTracer().reportMessageSent(numMessages);
                          }
                      }
                  });
    } else {
        // 如果发送的字节为空，则不会影响流控，仅仅发送
        writeQueue.enqueue(new SendGrpcFrameCommand(transportState(), bytebuf, endOfStream), flush);
    }
}
``` 

#### 半关闭

- io.grpc.internal.AbstractClientStream#halfClose

从客户端关闭流，关闭后客户端不能再发送消息，但是可以接收

```java
public final void halfClose() {
    if (!transportState().isOutboundClosed()) {
        transportState().setOutboundClosed();
        // 输出已经到达消息结尾
        endOfMessages();
    }
}
```

- io.grpc.internal.AbstractStream#endOfMessages

```java
protected final void endOfMessages() {
    framer().close();
}
```

- io.grpc.internal.MessageFramer#close

调用 Framer，释放缓冲区， 提交流；最终还是通过 Netty，将关闭流的帧写入，发送给服务端

```java
public void close() {
  if (!isClosed()) {
    closed = true;
    // With the current code we don't expect readableBytes > 0 to be possible here, added
    // defensively to prevent buffer leak issues if the framer code changes later.
    if (buffer != null && buffer.readableBytes() == 0) {
      releaseBuffer();
    }
    commitToSink(true, true);
  }
}
```

#### 获取返回结果

在 `io.grpc.stub.ClientCalls#blockingUnaryCall` 方法中，调用完 `futureUnaryCall` 方法后，会返回 `ListenableFuture`用于监听返回结果

- io.grpc.stub.ClientCalls#blockingUnaryCall

会不断的循环，监听线程池返回的结果
```java
ListenableFuture<RespT> responseFuture = futureUnaryCall(call, req);
while (!responseFuture.isDone()) {
  try {
    executor.waitAndDrain();
  } catch (InterruptedException e) {
    interrupt = true;
    call.cancel("Thread interrupted", e);
  }
}
return getUnchecked(responseFuture);
```

当 Server 端返回响应内容时，会调用监听器的 `messagesAvailable` 方法，从响应的流中解析响应内容

- io.grpc.internal.ClientCallImpl.ClientStreamListenerImpl#messagesAvailable

```java
try {
  InputStream message;
  while ((message = producer.next()) != null) {
    try {
	  // 将消息流解析为响应对象，并传递给 Future
      observer.onMessage(method.parseResponse(message));
    } catch (Throwable t) {
      GrpcUtil.closeQuietly(message);
      throw t;
    }
    message.close();
  }
} catch (Throwable t) {
  GrpcUtil.closeQuietly(producer);
  Status status = Status.CANCELLED.withCause(t).withDescription("Failed to read message.");
  stream.cancel(status);
  close(status, new Metadata());
}
```

- io.grpc.stub.ClientCalls.UnaryStreamToFuture#onMessage

为 Future 对象设置值

```java
public void onMessage(RespT value) {
  if (this.value != null) {
    throw Status.INTERNAL.withDescription("More than one value received for unary call")
                         .asRuntimeException();
  }
  this.value = value;
}
```

- io.grpc.stub.ClientCalls#getUnchecked

返回 Future 的值

```java
private static <V> V getUnchecked(Future<V> future) {
  try {
    return future.get();
  } catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw Status.CANCELLED
            .withDescription("Thread interrupted")
            .withCause(e)
            .asRuntimeException();
  } catch (ExecutionException e) {
    throw toStatusRuntimeException(e.getCause());
  }
}
```