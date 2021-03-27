---
title: Thrift 客户端异步请求
date: 2021-02-20 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrift 客户端异步请求

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

### 客户端

异步客户端调用使用 `AsyncClient`，传输层使用 `TNonblockingSocket`；需要实现 `AsyncMethodCallback`作为回调，用于处理请求结果

```java
public class AsyncClient {

    public static void main(String[] args) throws InterruptedException {

        try {
            // 构建异步客户端
            TAsyncClientManager clientManager = new TAsyncClientManager();
            TProtocolFactory protocolFactory = new TBinaryProtocol.Factory();

            HelloService.AsyncClient.Factory factory = new HelloService.AsyncClient.Factory(clientManager, protocolFactory);

            TNonblockingTransport nonblockingTransport = new TNonblockingSocket("localhost", 9090);
            HelloService.AsyncClient asyncClient = factory.getAsyncClient(nonblockingTransport);

            // 异步回调
            AsyncMethodCallback<HelloResponse> callback = new AsyncMethodCallback<HelloResponse>() {
                @Override
                public void onComplete(HelloResponse response) {
                    log.info("响应结果: {}", response.getMessage());
                }

                @Override
                public void onError(Exception exception) {
                    log.error("请求失败: {}", exception.getMessage(), exception);
                }
            };

            // 构建请求
            HelloMessage request = new HelloMessage();
            request.setMessage("Async Thrift");

            // 调用
            asyncClient.sayHello(request, callback);

        } catch (TException | IOException e) {
            e.printStackTrace();
        }
        Thread.sleep(3_000);
    }
}
```

## 请求处理流程

### 构建 Client 和回调

#### 1. 构建 Client

异步的客户端由抽象类 `TAsyncClient` 定义，实现类继承了 `TAsyncClient`，同时实现了 `AsyncIface`；
在构建其 Factory 时需要三个参数，分别是 `TAsyncClientManager`，用于管理调用请求的所有流程；和 `TProtocolFactory`，用于获取协议；还有 `TNonblockingTransport`，用于底层的传输，必须是非阻塞的

```java
// 构建异步客户端
TAsyncClientManager clientManager = new TAsyncClientManager();
TProtocolFactory protocolFactory = new TBinaryProtocol.Factory();

HelloService.AsyncClient.Factory factory = new HelloService.AsyncClient.Factory(clientManager, protocolFactory);

TNonblockingTransport nonblockingTransport = new TNonblockingSocket("localhost", 9090);
HelloService.AsyncClient asyncClient = factory.getAsyncClient(nonblockingTransport);
```

#### 2. 构建回调

回调由 `AsyncMethodCallback` 接口定义，有两个方法，分别是用于处理正常返回的 `onComplete` 和处理异常结果的 `onError`

```java
AsyncMethodCallback<HelloResponse> callback = new AsyncMethodCallback<HelloResponse>() {
    @Override
    public void onComplete(HelloResponse response) {
        log.info("响应结果: {}", response.getMessage());
    }

    @Override
    public void onError(Exception exception) {
        log.error("请求失败: {}", exception.getMessage(), exception);
    }
};
```

### 执行请求

#### 发起请求

调用接口时需要两个参数，请求内容和响应回调

```java
HelloMessage request = new HelloMessage();
request.setMessage("Async Thrift");

// 调用
asyncClient.sayHello(request, callback);
```

- io.github.helloworlde.thrift.HelloService.AsyncClient#sayHello

会构建 `sayHello_call`，该类是 `TAsyncMethodCall` 的实现类，支持写入消息和获取结果，然后将该实例作为参数调用 `org.apache.thrift.async.TAsyncClientManager#call` 方法

```java
public void sayHello(HelloMessage request, org.apache.thrift.async.AsyncMethodCallback<HelloResponse> resultHandler) throws org.apache.thrift.TException {
    checkReady();
    sayHello_call method_call = new sayHello_call(request, resultHandler, this, ___protocolFactory, ___transport);
    this.___currentMethod = method_call;
    ___manager.call(method_call);
}
```

- org.apache.thrift.async.TAsyncClientManager#call

初始化请求，将请求加入到队列中，然后唤醒 Selector 执行

```java
public void call(TAsyncMethodCall method) throws TException {
    if (!isRunning()) {
        throw new TException("SelectThread is not running");
    }
    // 初始化缓冲区
    method.prepareMethodCall();
    // 添加到待执行方法中，由 SelectThread 处理
    pendingCalls.add(method);
    selectThread.getSelector().wakeup();
}
```

- org.apache.thrift.async.TAsyncMethodCall#prepareMethodCall

执行初始化请求，，将请求消息写入到 Protocol 中，然后封装为 ByteBuffer 

```java
protected void prepareMethodCall() throws TException {
    TMemoryBuffer memoryBuffer = new TMemoryBuffer(INITIAL_MEMORY_BUFFER_SIZE);
    TProtocol protocol = protocolFactory.getProtocol(memoryBuffer);
    write_args(protocol);

    int length = memoryBuffer.length();
    frameBuffer = ByteBuffer.wrap(memoryBuffer.getArray(), 0, length);

    TFramedTransport.encodeFrameSize(length, sizeBufferArray);
    sizeBuffer = ByteBuffer.wrap(sizeBufferArray);
}
```

- io.github.helloworlde.thrift.HelloService.AsyncClient.sayHello_call#write_args

将消息写入到流中

```java
public void write_args(org.apache.thrift.protocol.TProtocol prot) throws org.apache.thrift.TException {
    prot.writeMessageBegin(new org.apache.thrift.protocol.TMessage("sayHello", org.apache.thrift.protocol.TMessageType.CALL, 0));
    sayHello_args args = new sayHello_args();
    args.setRequest(request);
    args.write(prot);
    prot.writeMessageEnd();
}
```

- org.apache.thrift.async.TAsyncClientManager.SelectThread#run

由 `SelectThread` 处理请求，在执行时，会先修改 `TAsyncMethodCall` 的状态，如果需要连接，则执行建立连接；如果需要写入或者读取则执行写入或者读取；然后会检查请求是否超时，如果没有超时，则开始执行待调用的方法

```java
public void run() {
    while (running) {
        try {
            try {
                if (timeoutWatchSet.size() == 0) {
                    selector.select();
                } else {
                    long nextTimeout = timeoutWatchSet.first().getTimeoutTimestamp();
                    long selectTime = nextTimeout - System.currentTimeMillis();
                    if (selectTime > 0) {
                        selector.select(selectTime);
                    } else {
                        // Next timeout is now or in past, select immediately so we can time out
                        selector.selectNow();
                    }
                }
            } catch (IOException e) {
                LOGGER.error("Caught IOException in TAsyncClientManager!", e);
            }
            // 过渡状态
            transitionMethods();
            // 检查是否超时
            timeoutMethods();
            // 开始待执行的方法
            startPendingMethods();
        } catch (Exception exception) {
            LOGGER.error("Ignoring uncaught exception in SelectThread", exception);
        }
    }

    try {
        selector.close();
    } catch (IOException ex) {
        LOGGER.warn("Could not close selector. This may result in leaked resources!", ex);
    }
}
```

- org.apache.thrift.async.TAsyncClientManager.SelectThread#startPendingMethods

从队列中获取待执行的请求，然后注册选择器，根据状态进行相应的操作

```java
private void startPendingMethods() {
    TAsyncMethodCall methodCall;
    while ((methodCall = pendingCalls.poll()) != null) {
        try {
            // 注册 Selector，开始第一个状态，既可以用于连接，也可以用于写入
            methodCall.start(selector);

            // 如果有超时则添加超时
            TAsyncClient client = methodCall.getClient();
            if (client.hasTimeout() && !client.hasError()) {
                timeoutWatchSet.add(methodCall);
            }
        } catch (Exception exception) {
            LOGGER.warn("Caught exception in TAsyncClientManager!", exception);
            methodCall.onError(exception);
        }
    }
}
```

- org.apache.thrift.async.TAsyncMethodCall#start

根据状态向 Transport 注册相应的操作

```java
void start(Selector sel) throws IOException {
    SelectionKey key;
    // 如果 Transport 是开启的，则修改状态为 WRITING_REQUEST_SIZE，注册写操作
    if (transport.isOpen()) {
        state = State.WRITING_REQUEST_SIZE;
        key = transport.registerSelector(sel, SelectionKey.OP_WRITE);
    } else {
        // Transport 状态不是开启的，修改状态为 CONNECTING，注册连接操作
        state = State.CONNECTING;
        key = transport.registerSelector(sel, SelectionKey.OP_CONNECT);

        // non-blocking connect can complete immediately,
        // in which case we should not expect the OP_CONNECT
        // 开启连接后注册写入
        if (transport.startConnect()) {
            registerForFirstWrite(key);
        }
    }

    key.attach(this);
}
```


- org.apache.thrift.async.TAsyncMethodCall#doWritingRequestBody

在 `org.apache.thrift.async.TAsyncMethodCall#transition` 执行建立连接，写入请求头后开始写入请求体；当写入完成后，，会根据请求类型做相应操作；如果是 oneway，则执行请求回调流程并完成请求；如果不是，则修改状态，等待读取

```java
private void doWritingRequestBody(SelectionKey key) throws IOException {
    if (transport.write(frameBuffer) < 0) {
        throw new IOException("Write call frame failed");
    }
    if (frameBuffer.remaining() == 0) {
        if (isOneway) {
            cleanUpAndFireCallback(key);
        } else {
            state = State.READING_RESPONSE_SIZE;
            sizeBuffer.rewind();  // Prepare to read incoming frame size
            key.interestOps(SelectionKey.OP_READ);
        }
    }
}
```

#### 处理响应

在请求发送完成后，`SelectionKey`的状态被修改为 `READING_RESPONSE_SIZE`，当接收到服务端的响应后，会先读取响应大小，然后将状态修改为 `READING_RESPONSE_BODY`，然后读取响应内容

- org.apache.thrift.async.TAsyncMethodCall#doReadingResponseBody

会读取所有的响应内容到缓冲区，然后执行请求完成流程

```java
private void doReadingResponseBody(SelectionKey key) throws IOException {
    if (transport.read(frameBuffer) < 0) {
        throw new IOException("Read call frame failed");
    }
    if (frameBuffer.remaining() == 0) {
        cleanUpAndFireCallback(key);
    }
}
```

- org.apache.thrift.async.TAsyncMethodCall#cleanUpAndFireCallback

修改请求状态，释放 `SelectionKey`，然后获取响应结果，执行回调

```java
private void cleanUpAndFireCallback(SelectionKey key) {
    state = State.RESPONSE_READ;
    key.interestOps(0);
    // this ensures that the TAsyncMethod instance doesn't hang around
    key.attach(null);
    try {
        T result = this.getResult();
        client.onComplete();
        callback.onComplete(result);
    } catch (Exception e) {
        key.cancel();
        onError(e);
    }
}
```

- io.github.helloworlde.thrift.HelloService.AsyncClient.sayHello_call#getResult

```java
public HelloResponse getResult() throws org.apache.thrift.TException {
    if (getState() != org.apache.thrift.async.TAsyncMethodCall.State.RESPONSE_READ) {
        throw new java.lang.IllegalStateException("Method call not finished!");
    }
    org.apache.thrift.transport.TMemoryInputTransport memoryTransport = new org.apache.thrift.transport.TMemoryInputTransport(getFrameBuffer().array());
    org.apache.thrift.protocol.TProtocol prot = client.getProtocolFactory().getProtocol(memoryTransport);
    return (new Client(prot)).recv_sayHello();
}
```

- io.github.helloworlde.thrift.HelloService.Client#recv_sayHello

会先构建响应对象，然后获取结果内容，作为结果返回

```java
public HelloResponse recv_sayHello() throws org.apache.thrift.TException {
    sayHello_result result = new sayHello_result();
    receiveBase(result, "sayHello");
    if (result.isSetSuccess()) {
        return result.success;
    }
    throw new org.apache.thrift.TApplicationException(org.apache.thrift.TApplicationException.MISSING_RESULT, "sayHello failed: unknown result");
}
```

- org.apache.thrift.TServiceClient#receiveBase

从 `TProtocol` 中读取响应内容，根据响应的结果，如果是正常则由相应类读取并解析为其实例，如果异常则抛出

```java
protected void receiveBase(TBase<?, ?> result, String methodName) throws TException {
    // 读取消息
    TMessage msg = iprot_.readMessageBegin();
    // 如果是异常，则读取异常并抛出
    if (msg.type == TMessageType.EXCEPTION) {
        TApplicationException x = new TApplicationException();
        x.read(iprot_);
        iprot_.readMessageEnd();
        throw x;
    }
    // 如果请求序号不匹配，则抛出异常
    if (msg.seqid != seqid_) {
        throw new TApplicationException(TApplicationException.BAD_SEQUENCE_ID,
                String.format("%s failed: out of sequence response: expected %d but got %d", methodName, seqid_, msg.seqid));
    }
    // 读取响应内容
    result.read(iprot_);
    iprot_.readMessageEnd();
}
```

- org.apache.thrift.async.AsyncMethodCallback#onComplete

最终执行回调

```java
public void onComplete(HelloResponse response) {
    log.info("响应结果: {}", response.getMessage());
}
```

## 参考文档

- [helloworlde/thrift-java-sample](https://github.com/helloworlde/thrift-java-sample)