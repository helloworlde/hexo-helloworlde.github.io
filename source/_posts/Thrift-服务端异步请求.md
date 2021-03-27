---
title: Thrift 服务端异步请求
date: 2021-02-01 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrift 服务端异步请求

## 实现

### IDL 

- helloworld.thrift

```thrift
namespace java io.github.helloworlde.thrift

struct HelloMessage {
    1: required string message,
}

struct HelloResponse {
    1: required string message,
}

service HelloService {
    HelloResponse sayHello(1: HelloMessage request);
}
```

### 服务端

- Server

```java
@Slf4j
public class AsyncServer {

    @SneakyThrows
    public static void main(String[] args) {

        HelloServiceAsyncImpl helloService = new HelloServiceAsyncImpl();
        HelloService.AsyncProcessor<HelloService.AsyncIface> helloServiceProcessor = new HelloService.AsyncProcessor<>(helloService);

        TNonblockingServerTransport transport = new TNonblockingServerSocket(9090);

        // 配置参数以及处理器
        TThreadedSelectorServer.Args serverArgs = new TThreadedSelectorServer.Args(transport)
                .selectorThreads(4)
                .workerThreads(10)
                .acceptQueueSizePerThread(20)
                .processor(helloServiceProcessor);

        TServer server = new TThreadedSelectorServer(serverArgs);
        server.serve();
    }
}
```

- 实现

实现类需要实现 `AsyncIface` 接口，方法定义中会有一个 `AsyncMethodCallback`，用于处理响应回调

```java
@Slf4j
public class HelloServiceAsyncImpl implements HelloService.AsyncIface {

    @Override
    public void sayHello(HelloMessage request, AsyncMethodCallback<HelloResponse> resultHandler) throws TException {
        String message = request.getMessage();
        log.info("接收到请求: {}", message);

        HelloResponse response = new HelloResponse();
        response.setMessage("Hello " + message);

        resultHandler.onComplete(response);
    }
}
```

## 请求处理流程

Server 端同步与异步处理的流程区别在于使用的 `TProcessor` 不同；同步使用 `TProcessor`，异步使用 `TAsyncProcessor`；除此之外，其他的流程与使用 NIO 的同步处理没有区别

### 执行请求

- org.apache.thrift.server.AbstractNonblockingServer.AsyncFrameBuffer#invoke

在处理读取事件时，会将 AsyncFrameBuffer 包装为 Runnable，提交给线程池执行；最终由 AsyncFrameBuffer 处理 
会获取  Processor，然后调用  process 方法进行处理

```java
public void invoke() {
    // 重置 Transport
    frameTrans_.reset(buffer_.array());
    response_.reset();

    try {
        // 触发事件
        if (eventHandler_ != null) {
            eventHandler_.processContext(context_, inTrans_, outTrans_);
        }
        // 调用处理器处理
        ((TAsyncProcessor) processorFactory_.getProcessor(inTrans_)).process(this);
        return;
    } catch (TException te) {
        LOGGER.warn("Exception while invoking!", te);
    } catch (Throwable t) {
        LOGGER.error("Unexpected throwable while invoking!", t);
    }
    // This will only be reached when there is a throwable.
    // 修改状态
    state_ = FrameBufferState.AWAITING_CLOSE;
    requestSelectInterestChange();
}
```

- org.apache.thrift.TBaseAsyncProcessor#process(org.apache.thrift.server.AbstractNonblockingServer.AsyncFrameBuffer)

会读取消息，然后根据方法名称获取处理的类，然后获取调用回调，将请求信息和回调作为参数，调用处理函数处理请求

```java
public void process(final AsyncFrameBuffer fb) throws TException {

    final TProtocol in = fb.getInputProtocol();
    final TProtocol out = fb.getOutputProtocol();

    // 读取消息
    final TMessage msg = in.readMessageBegin();
    // 获取处理函数
    AsyncProcessFunction fn = processMap.get(msg.name);
    // 获取空参数
    TBase args = fn.getEmptyArgsInstance();

    // 读取参数
    args.read(in);
    in.readMessageEnd();

    // 如果是 oneway 调用，则完成
    if (fn.isOneway()) {
        fb.responseReady();
    }

    //start off processing function
    // 获取处理方法
    AsyncMethodCallback resultHandler = fn.getResultHandler(fb, msg.seqid);
    try {
        // 处理调用
        fn.start(iface, args, resultHandler);
    } catch (Exception e) {
        LOGGER.debug("Exception handling function", e);
        resultHandler.onError(e);
    }
    return;
}
```

- io.github.helloworlde.thrift.HelloService.AsyncProcessor.sayHello#start

```java
public void start(I iface, sayHello_args args, org.apache.thrift.async.AsyncMethodCallback<HelloResponse> resultHandler) 
throws org.apache.thrift.TException {
    iface.sayHello(args.request, resultHandler);
}
```

- io.github.helloworlde.thrift.HelloServiceAsyncImpl#sayHello

然后会调用实现类，执行具体的处理逻辑；在处理完成后需要调用回调的相应方法

```java
@Slf4j
public class HelloServiceAsyncImpl implements HelloService.AsyncIface {

    @Override
    public void sayHello(HelloMessage request, AsyncMethodCallback<HelloResponse> resultHandler) throws TException {
        String message = request.getMessage();
        log.info("接收到请求: {}", message);

        HelloResponse response = new HelloResponse();
        response.setMessage("Hello " + message);

        resultHandler.onComplete(response);
    }
}
```

### 返回响应

- org.apache.thrift.async.AsyncMethodCallback#onComplete

请求处理成功的回调，会将响应结果发送出去

```java
public void onComplete(HelloResponse o) {
    sayHello_result result = new sayHello_result();
    result.success = o;
    try {
        fcall.sendResponse(fb, result, org.apache.thrift.protocol.TMessageType.REPLY, seqid);
    } catch (org.apache.thrift.transport.TTransportException e) {
        _LOGGER.error("TTransportException writing to internal frame buffer", e);
        fb.close();
    } catch (java.lang.Exception e) {
        _LOGGER.error("Exception writing to internal frame buffer", e);
        onError(e);
    }
}
```

- org.apache.thrift.AsyncProcessFunction#sendResponse

将方法、消息类型，请求的 ID，响应内容按序写入，然后全部发送给传输层，由传输层发送给客户端；请求处理完成

```java
public void sendResponse(final AbstractNonblockingServer.AsyncFrameBuffer fb, final TSerializable result, final byte type, final int seqid) throws TException {
    TProtocol oprot = fb.getOutputProtocol();

    oprot.writeMessageBegin(new TMessage(getMethodName(), type, seqid));
    result.write(oprot);
    oprot.writeMessageEnd();
    oprot.getTransport().flush();

    fb.responseReady();
}
```

## 参考文档

- [helloworlde/thrift-java-sample](https://github.com/helloworlde/thrift-java-sample)