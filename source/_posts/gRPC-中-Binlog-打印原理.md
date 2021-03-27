---
title: gRPC 中 Binlog 打印原理
date: 2020-09-20 22:33:59
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 中 Binlog 打印原理

gRPC 支持将请求调用的参数、Header 等信息以二进制的方式输出到文件中

## 使用 

binlog 的依赖在 `grpc-services`中，所以需要有该依赖

- 创建 Channel 时指定

```java
BinaryLog binaryLog = BinaryLogs.createBinaryLog(new TempFileSink(), "*");
this.channel = ManagedChannelBuilder.forAddress(host, port)
                                    .usePlaintext()
                                    .setBinaryLog(binaryLog)
                                    .build();
```

在创建时，需要指定打印的方法，`*`代表打印所有的方法，具体指定可以参考 [Control Interface](https://github.com/helloworlde/proposal/blob/master/A16-binary-logging.md#control-interface)
也可以在创建时不指定参数，通过设置环境变量 `GRPC_BINARY_LOG_CONFIG=*`来指定需要打印的方法
如果需要指定文件的生成位置，可以重写`io.grpc.services.BinaryLogSink`，指定文件位置

## 实现

在方法调用时，会判断有没有设置 binlog 对象，如果有则会封装方法，添加处理器和监听器；然后重新创建 `ServerMethodDefinition`；通过二进制日志拦截器 `io.grpc.services.BinlogHelper#getClientInterceptor` 拦截请求并写入日志 


- io.grpc.internal.ServerImpl.ServerTransportListenerImpl#startCall

```java
    private <ReqT, RespT> ServerStreamListener startCall(ServerStream stream, String fullMethodName,
        ServerMethodDefinition<ReqT, RespT> methodDef, Metadata headers,
        Context.CancellableContext context, StatsTraceContext statsTraceCtx, Tag tag) {

      // 如果 binlog 不为空，即需要记录binlog，则添加请求监听器和方法处理器记录 binlog
      ServerMethodDefinition<?, ?> wMethodDef = binlog == null ? interceptedDef : binlog.wrapMethodDefinition(interceptedDef);

      return startWrappedCall(fullMethodName, wMethodDef, stream, headers, context, tag);
    }
```

- io.grpc.services.BinaryLogProvider#wrapMethodDefinition

```
  public final <ReqT, RespT> ServerMethodDefinition<?, ?> wrapMethodDefinition(ServerMethodDefinition<ReqT, RespT> oMethodDef) {
    // 根据方法获取二进制日志拦截器，如果没有该方法则不拦截
    ServerInterceptor binlogInterceptor = getServerInterceptor(oMethodDef.getMethodDescriptor().getFullMethodName());
    if (binlogInterceptor == null) {
      return oMethodDef;
    }

    MethodDescriptor<byte[], byte[]> binMethod = BinaryLogProvider.toByteBufferMethod(oMethodDef.getMethodDescriptor());
    // 包装方法，添加了处理器和监听器
    ServerMethodDefinition<byte[], byte[]> binDef = InternalServerInterceptors.wrapMethod(oMethodDef, binMethod);
    // 创建处理器
    ServerCallHandler<byte[], byte[]> binlogHandler =
            InternalServerInterceptors.interceptCallHandlerCreate(binlogInterceptor, binDef.getServerCallHandler());
    // 创建服务方法定义
    return ServerMethodDefinition.create(binMethod, binlogHandler);
  }
```

- io.grpc.services.BinlogHelper#getClientInterceptor

```
  public ClientInterceptor getClientInterceptor(final long callId) {
    return new ClientInterceptor() {
      boolean trailersOnlyResponse = true;

      @Override
      public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
              final MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, Channel next) {
        final String methodName = method.getFullMethodName();
        final String authority = next.authority();
        final Deadline deadline = min(callOptions.getDeadline(), Context.current().getDeadline());

        return new SimpleForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
          @Override
          public void start(final ClientCall.Listener<RespT> responseListener, Metadata headers) {
            final Duration timeout = deadline == null ? null
                    : Durations.fromNanos(deadline.timeRemaining(TimeUnit.NANOSECONDS));
            writer.logClientHeader(
                    seq.getAndIncrement(),
                    methodName,
                    authority,
                    timeout,
                    headers,
                    GrpcLogEntry.Logger.LOGGER_CLIENT,
                    callId,
                    /*peerAddress=*/ null);
            ClientCall.Listener<RespT> wListener =
                    new SimpleForwardingClientCallListener<RespT>(responseListener) {
                      @Override
                      public void onMessage(RespT message) {
                        writer.logRpcMessage(
                                seq.getAndIncrement(),
                                EventType.EVENT_TYPE_SERVER_MESSAGE,
                                method.getResponseMarshaller(),
                                message,
                                GrpcLogEntry.Logger.LOGGER_CLIENT,
                                callId);
                        super.onMessage(message);
                      }

                      @Override
                      public void onHeaders(Metadata headers) {
                        trailersOnlyResponse = false;
                        writer.logServerHeader(
                                seq.getAndIncrement(),
                                headers,
                                GrpcLogEntry.Logger.LOGGER_CLIENT,
                                callId,
                                getPeerSocket(getAttributes()));
                        super.onHeaders(headers);
                      }

                      @Override
                      public void onClose(Status status, Metadata trailers) {
                        SocketAddress peer = trailersOnlyResponse
                                ? getPeerSocket(getAttributes()) : null;
                        writer.logTrailer(
                                seq.getAndIncrement(),
                                status,
                                trailers,
                                GrpcLogEntry.Logger.LOGGER_CLIENT,
                                callId,
                                peer);
                        super.onClose(status, trailers);
                      }
                    };
            super.start(wListener, headers);
          }

          @Override
          public void sendMessage(ReqT message) {
            writer.logRpcMessage(
                    seq.getAndIncrement(),
                    EventType.EVENT_TYPE_CLIENT_MESSAGE,
                    method.getRequestMarshaller(),
                    message,
                    GrpcLogEntry.Logger.LOGGER_CLIENT,
                    callId);
            super.sendMessage(message);
          }

          @Override
          public void halfClose() {
            writer.logHalfClose(
                    seq.getAndIncrement(),
                    GrpcLogEntry.Logger.LOGGER_CLIENT,
                    callId);
            super.halfClose();
          }

          @Override
          public void cancel(String message, Throwable cause) {
            writer.logCancel(
                    seq.getAndIncrement(),
                    GrpcLogEntry.Logger.LOGGER_CLIENT,
                    callId);
            super.cancel(message, cause);
          }
        };
      }
    };
  }
```