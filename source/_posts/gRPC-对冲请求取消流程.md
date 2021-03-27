---
title: gRPC 对冲请求取消流程
date: 2021-02-20 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 对冲请求取消流程

当客户端接收到对冲请求集合中的一个完成时，会取消其他的请求，被取消的请求最终会提交一个 CancelClientStreamCommand，发送一个 RST_STEAM 请求；当服务端接受到这个流后，如果监听器还没有关闭，会执行取消上下文的操作，最终将这个请求取消

![grpc-hedging-request-cancel.svg](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-hedging-request-cancel.svg)


## 客户端

当客户端成功接收到响应会，会在 io.grpc.internal.RetriableStream.Sublistener#close 中将成功的流进行提交

- io.grpc.internal.RetriableStream#commit$CommitTask#run 

在提交时，会通过提交 CommitTask 将其他的流取消

```java
class CommitTask implements Runnable {
    @Override
    public void run() {
        // 遍历保存的枯竭的流，如果不是最后提交的流，则都取消
        for (Substream substream : savedDrainedSubstreams) {
            if (substream != winningSubstream) {
                substream.stream.cancel(CANCELLED_BECAUSE_COMMITTED);
            }
        }
        // 如果有重试中的，则取消
        if (retryFuture != null) {
            retryFuture.cancel(false);
        }
        // 如果有对冲中的，则取消
        if (hedgingFuture != null) {
            hedgingFuture.cancel(false);
        }

        // 将当前流从未提交的流中移除
        postCommit();
    }
}
```

- io.grpc.internal.AbstractClientStream#cancel

使用指定的原因取消流

```java
public final void cancel(Status reason) {
    Preconditions.checkArgument(!reason.isOk(), "Should not cancel with OK status");
    cancelled = true;
    abstractClientStreamSink().cancel(reason);
}
```

- io.grpc.netty.shaded.io.grpc.netty.NettyClientStream.Sink#cancel

提交取消流的指令

```java
public void cancel(Status status) {
    PerfMark.startTask("NettyClientStream$Sink.cancel");

    try {
        NettyClientStream.this.writeQueue.enqueue(new CancelClientStreamCommand(NettyClientStream.this.transportState(), status), true);
    } finally {
        PerfMark.stopTask("NettyClientStream$Sink.cancel");
    }

}
```

- io.grpc.netty.shaded.io.grpc.netty.NettyClientHandler#write

在执行写入消息时，写入取消指令

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    if (msg instanceof CreateStreamCommand) {
        this.createStream((CreateStreamCommand)msg, promise);
    } else if (msg instanceof SendGrpcFrameCommand) {
        this.sendGrpcFrame(ctx, (SendGrpcFrameCommand)msg, promise);
    } else if (msg instanceof CancelClientStreamCommand) {
        this.cancelStream(ctx, (CancelClientStreamCommand)msg, promise);
    } else if (msg instanceof SendPingCommand) {
        this.sendPingFrame(ctx, (SendPingCommand)msg, promise);
    } else if (msg instanceof GracefulCloseCommand) {
        this.gracefulClose(ctx, (GracefulCloseCommand)msg, promise);
    } else if (msg instanceof ForcefulCloseCommand) {
        this.forcefulClose(ctx, (ForcefulCloseCommand)msg, promise);
    } else {
        if (msg != NOOP_MESSAGE) {
            throw new AssertionError("Write called for unexpected type: " + msg.getClass().getName());
        }

        ctx.write(Unpooled.EMPTY_BUFFER, promise);
    }

}
```

- io.grpc.netty.shaded.io.grpc.netty.NettyClientHandler#cancelStream

执行取消命令的写入，在 transportReportStatus 会提交关闭监听器的指令，如果停止投递，同时也会选择执行或者延迟执行关闭帧
如果流存在，则会发送一个新的 RST_STREAM 请求，该请求表示当前流错误，错误状态为 CANCEL，即值为 8

```java
private void cancelStream(ChannelHandlerContext ctx, CancelClientStreamCommand cmd, ChannelPromise promise) {
    TransportState stream = cmd.stream();
    try {
        Status reason = cmd.reason();
        if (reason != null) {
            stream.transportReportStatus(reason, true, new Metadata());
        }

        if (!cmd.stream().isNonExistent()) {
            this.encoder().writeRstStream(ctx, stream.id(), Http2Error.CANCEL.code(), promise);
        } else {
            promise.setSuccess();
        }
    } finally {
        PerfMark.stopTask("NettyClientHandler.cancelStream", stream.tag());
    }

}
```

## 服务端

- io.grpc.netty.shaded.io.grpc.netty.NettyServerHandler.FrameListener#onRstStreamRead

接收到的请求中，errorCode 为 8，代表请求被取消

```java
public void onRstStreamRead(ChannelHandlerContext ctx, int streamId, long errorCode) throws Http2Exception {
    if (NettyServerHandler.this.keepAliveManager != null) {
        NettyServerHandler.this.keepAliveManager.onDataReceived();
    }

    NettyServerHandler.this.onRstStreamRead(streamId, errorCode);
}
```

- io.grpc.netty.shaded.io.grpc.netty.NettyServerHandler#onRstStreamRead

然后会在 NettyServerHandler 中根据 streamId 获取流，如果流存在，则会以 CANCELLED 状态取消当前请求
需要注意的是，如果接收到这个请求时流已经完成被清除，则可能无法处理，请求会以 OK 状态完成

```java
private void onRstStreamRead(int streamId, long errorCode) throws Http2Exception {
    try {
        TransportState stream = this.serverStream(this.connection().stream(streamId));
        if (stream != null) {
            try {
                stream.transportReportStatus(Status.CANCELLED.withDescription("RST_STREAM received for code " + errorCode));
            }
        }
    } catch (Throwable var9) {
        logger.log(Level.WARNING, "Exception in onRstStreamRead()", var9);
        throw this.newStreamException(streamId, var9);
    }
}
```

- io.grpc.internal.AbstractServerStream.TransportState#transportReportStatus

如果解帧器已经关闭，则使用取消状态关闭监听器

```java
public final void transportReportStatus(final Status status) {
    Preconditions.checkArgument(!status.isOk(), "status must not be OK");
    // 如果解帧器关闭，则关闭监听器
    if (deframerClosed) {
        deframerClosedTask = null;
        closeListener(status);
    } else {
        // 如果解帧器还没有关闭，则创建关闭监听器的任务，并立即关闭解帧器
        deframerClosedTask = new Runnable() {
            @Override
            public void run() {
                closeListener(status);
            }
        };
        immediateCloseRequested = true;
        closeDeframer(true);
    }
}
```

- io.grpc.internal.AbstractServerStream.TransportState#closeListener

使用指定状态关闭监听器

```java
private void closeListener(Status newStatus) {
    Preconditions.checkState(!newStatus.isOk() || closedStatus != null);
    // 如果监听器没有关闭，则根据状态操作
    if (!listenerClosed) {
        listenerClosed = true;
        // 通知流的状态不可以再使用
        onStreamDeallocated();
        // 使用指定状态关闭监听器
        listener().closed(newStatus);
    }
}
```

- io.grpc.internal.ServerImpl.JumpToApplicationThreadServerStreamListener#closedInternal

使用 CANCELLED 状态，执行 ContextCloser 任务，取消上下文；然后提交 Closed 任务，取消
监听器

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
            try {
                // 调用监听器的关闭事件
                getListener().closed(status);
            }
        }
    }

    callExecutor.execute(new Closed());
}
```

- io.grpc.internal.ServerImpl.ContextCloser#run

执行提交的 ContextCloser 任务，取消上下文

```java
public void run() {
    // 执行时使用指定异常关闭 Context
    context.cancel(cause);
}
```

- io.grpc.Context.CancellableContext#cancel

执行 Context 取消事件，会修改 Context 的状态，取消等待的 deadline 任务，然后会通知并清除监听器

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

- io.grpc.Context.CancellableContext#notifyAndClearListeners

会通知当前的监听器进行取消，默认由两个监听器，一个是`CancellationListener`，用于取消当前上下文，另一个是 `ServerStreamCancellationListener` ，用于取消流；如果有其他的监听器，还会通知其他监听器取消，并移除监听器

```java
private void notifyAndClearListeners() {
    ArrayList<ExecutableListener> tmpListeners;
    CancellationListener tmpParentListener;
    synchronized (this) {
        // 如果没有监听器则返回
        if (listeners == null) {
            return;
        }
        tmpParentListener = parentListener;
        parentListener = null;
        tmpListeners = listeners;
        listeners = null;
    }
    // 在取消之前先通知事件，优先通知当前上下文
    for (ExecutableListener tmpListener : tmpListeners) {
        if (tmpListener.context == this) {
            tmpListener.deliver();
        }
    }
    // 通知其他的上下文
    for (ExecutableListener tmpListener : tmpListeners) {
        if (!(tmpListener.context == this)) {
            tmpListener.deliver();
        }
    }
    // 移除引用的监听器
    if (cancellableAncestor != null) {
        cancellableAncestor.removeListener(tmpParentListener);
    }
}
```

- io.grpc.internal.ServerCallImpl.ServerStreamListenerImpl#ServerStreamListenerImpl

最终会调用构造 ServerStreamListenerImpl 时添加的 Context.CancellationListener 的 cancelled 方法，将 ServerCallImpl 的 cancelled 状态改为 true

```java
this.context.addListener(new Context.CancellationListener() {
    @Override
    public void cancelled(Context context) {
        ServerStreamListenerImpl.this.call.cancelled = true;
    }
}, MoreExecutors.directExecutor());
```

- io.grpc.internal.ServerImpl.ServerTransportListenerImpl#StreamCreated$ServerStreamCancellationListener#cancelled

执行创建流时添加的流取消监听器，如果没有异常信息，则会使用 `"io.grpc.Context was cancelled without error"`作为描述，更新状态

```java
public void cancelled(Context context) {
    Status status = statusFromCancelled(context);
    if (DEADLINE_EXCEEDED.getCode().equals(status.getCode())) {
        // This should rarely get run, since the client will likely cancel the stream
        // before the timeout is reached.
        stream.cancel(status);
    }
}
```


- io.grpc.internal.ServerCallImpl.ServerStreamListenerImpl#closedInternal

在执行 OnClosed 任务时，会使用 CANCELLED 状态，触发 ServerCall.Listener 的 onCanncel 事件，如果有取消任务，会执行取消任务

另外，无论请求成功与否，都会执行 `context.cancel(null)`，通知 `notifyAndClearListeners`取消上下文监听器和流监听器，然后移除监听器

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

