---
title: Thrfit 服务端请求处理流程
date: 2021-02-20 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrfit 服务端请求处理流程

使用同步的非阻塞的服务端的请求处理流程

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

### 服务端实现

使用 `TThreadedSelectorServer` 作为服务端，支持接收连接，处理 IO 事件，执行请求由不同的线程实现；底层连接使用 `ServerSocket`

```java
public class NonblockingServer {

    @SneakyThrows
    public static void main(String[] args) {

        HelloServiceImpl helloService = new HelloServiceImpl();
        HelloService.Processor<HelloService.Iface> helloServiceProcessor = new HelloService.Processor<>(helloService);

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

## 请求处理流程 


### 1. 启动 Server

```java
TServer server = new TThreadedSelectorServer(serverArgs);
server.serve();
```

-  org.apache.thrift.server.AbstractNonblockingServer#serve

启动 Server，启动用于连接的线程 `AcceptThread` 和用于处理 IO 事件的多个线程  `SelectorThread`；然后开始监听 IO 事件，由线程池处理请求

```java
public void serve() {
    // 启动
    if (!startThreads()) {
        return;
    }
    // 开始监听
    if (!startListening()) {
        return;
    }
    // 修改状态
    setServing(true);
    // 阻塞直到关闭
    waitForShutdown();
    setServing(false);
    // 停止监听器
    stopListening();
}
```

- org.apache.thrift.server.TThreadedSelectorServer#startThreads

启动用于连接的线程 `AcceptThread` 和用于处理 IO 事件的多个线程  `SelectorThread` 

```java
protected boolean startThreads() {
    try {
        // 创建选择线程，并添加到集合中
        for (int i = 0; i < args.selectorThreads; ++i) {
            selectorThreads.add(new SelectorThread(args.acceptQueueSizePerThread));
        }
        // 创建处理连接的负载均衡， 创建处理连接的线程
        acceptThread = new AcceptThread((TNonblockingServerTransport) serverTransport_, createSelectorThreadLoadBalancer(selectorThreads));

        // 启动选择线程
        for (SelectorThread thread : selectorThreads) {
            thread.start();
        }
        // 启动连接的线程
        acceptThread.start();
        return true;
    } catch (IOException e) {
        LOGGER.error("Failed to start threads!", e);
        return false;
    }
}
```

- org.apache.thrift.transport.TNonblockingServerSocket#listen

开始监听

```java
public void listen() throws TTransportException {
    // Make sure not to block on accept
    if (serverSocket_ != null) {
        try {
            serverSocket_.setSoTimeout(0);
        } catch (SocketException sx) {
            LOGGER.error("Socket exception while setting socket timeout", sx);
        }
    }
}
```

### 2. 处理连接事件

- org.apache.thrift.server.TThreadedSelectorServer.AcceptThread#run

连接事件由 `AcceptThread` 线程独立处理；会循环监听 `Selector`事件，当有新的连接事件时，会建立连接

```java
public void run() {
    try {
        if (eventHandler_ != null) {
            // 通知 Server 开始启动
            eventHandler_.preServe();
        }

        while (!stopped_) {
            // 选择处理连接
            select();
        }
    } finally {
        acceptSelector.close();
        TThreadedSelectorServer.this.stop();
    }
}
```

- org.apache.thrift.server.TThreadedSelectorServer.AcceptThread#select

会不断从 `Selector`获取事件，判断如果是 accept 事件，则处理，并建立连接

```java
private void select() {
    try {
        // 等待连接事件
        acceptSelector.select();

        // 处理接收到的事件
        Iterator<SelectionKey> selectedKeys = acceptSelector.selectedKeys().iterator();
        while (!stopped_ && selectedKeys.hasNext()) {
            SelectionKey key = selectedKeys.next();
            selectedKeys.remove();

            if (!key.isValid()) {
                continue;
            }

            if (key.isAcceptable()) {
                // 建立连接
                handleAccept();
            } else {
                LOGGER.warn("Unexpected state in select! " + key.interestOps());
            }
        }
    } catch (IOException e) {
        LOGGER.warn("Got an IOException while selecting!", e);
    }
}
```

- org.apache.thrift.server.TThreadedSelectorServer.AcceptThread#handleAccept

会通过底层的 `ServerSocketChannel` 建立连接，然后将这个连接添加到 `SelectorThread` 的队列中，由 `SelectorThread`处理 IO 事件

```java
private void handleAccept() {
    // 建立连接
    final TNonblockingTransport client = doAccept();
    if (client != null) {
        // 将连接传递给选择线程
        final SelectorThread targetThread = threadChooser.nextThread();

        // 如果策略是尽快建立连接，则添加到处理的队列中
        if (args.acceptPolicy == Args.AcceptPolicy.FAST_ACCEPT || invoker == null) {
            doAddAccept(targetThread, client);
        } else {
            try {
                // 如果是 FAIR_ACCEPT，则提交异步任务进行添加
                invoker.submit(new Runnable() {
                    public void run() {
                        doAddAccept(targetThread, client);
                    }
                });
            } catch (RejectedExecutionException rx) {
                LOGGER.warn("ExecutorService rejected accept registration!", rx);
                // close immediately
                client.close();
            }
        }
    }
}
```

- org.apache.thrift.transport.TNonblockingServerSocket#acceptImpl

建立连接，返回新的 `TNonblockingSocket`

```java
protected TNonblockingSocket acceptImpl() throws TTransportException {
    if (serverSocket_ == null) {
        throw new TTransportException(TTransportException.NOT_OPEN, "No underlying server socket.");
    }
    try {
        // 接受连接
        SocketChannel socketChannel = serverSocketChannel.accept();
        if (socketChannel == null) {
            return null;
        }

        // 使用 Channel 构建 Socket
        TNonblockingSocket tsocket = new TNonblockingSocket(socketChannel);
        tsocket.setTimeout(clientTimeout_);
        return tsocket;
    } catch (IOException iox) {
        throw new TTransportException(iox);
    }
}
```

- org.apache.thrift.server.TThreadedSelectorServer.SelectorThread#addAcceptedConnection

将连接添加到 `SelectorThread`的队列中，由 `SelectorThread`处理 IO 事件 

```java
public boolean addAcceptedConnection(TNonblockingTransport accepted) {
    try {
        // 放入队列中
        acceptedQueue.put(accepted);
    } catch (InterruptedException e) {
        LOGGER.warn("Interrupted while adding accepted connection!", e);
        return false;
    }
    selector.wakeup();
    return true;
}
```

### 3. 处理 IO 事件

IO  事件由 `SelectorThread` 处理

- org.apache.thrift.server.TThreadedSelectorServer.SelectorThread#run

轮询读取事件，如果是 IO 事件，则分别处理；如果是新的连接，则注册 `Selector`

```java
public void run() {
    try {
        while (!stopped_) {
            // 选择读取或写入事件
            select();
            // 处理新的连接
            processAcceptedConnections();
            // 改变需要改变的状态
            processInterestChanges();
        }

        // 如果停止了，则清理选择
        for (SelectionKey selectionKey : selector.keys()) {
            cleanupSelectionKey(selectionKey);
        }
    } catch (Throwable t) {
        LOGGER.error("run() on SelectorThread exiting due to uncaught error", t);
    } finally {
        // 关闭
        selector.close();
        TThreadedSelectorServer.this.stop();
    }
}
```

- org.apache.thrift.server.TThreadedSelectorServer.SelectorThread#select

处理 IO 事件，根据事件类型分别处理读取或者写入

```java
private void select() {
    try {
        // 获取事件
        doSelect();

        Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();
        while (!stopped_ && selectedKeys.hasNext()) {
            SelectionKey key = selectedKeys.next();
            selectedKeys.remove();

            // 如果无效则跳过
            if (!key.isValid()) {
                cleanupSelectionKey(key);
                continue;
            }

            if (key.isReadable()) {
                // 如果是读取则处理读取事件
                handleRead(key);
            } else if (key.isWritable()) {
                // 如果是写入则处理写入
                handleWrite(key);
            } else {
                LOGGER.warn("Unexpected state in select! " + key.interestOps());
            }
        }
    } catch (IOException e) {
        LOGGER.warn("Got an IOException while selecting!", e);
    }
}
```


#### 处理读取事件

- org.apache.thrift.server.AbstractNonblockingServer.AbstractSelectThread#handleRead

在处理读取事件时，会读取整个帧，当完全读取时，会调用 `requestInvoke` 方法，通过线程池处理请求

```java
protected void handleRead(SelectionKey key) {
    // 获取帧
    FrameBuffer buffer = (FrameBuffer) key.attachment();
    // 如果没有可读取的，则清理
    if (!buffer.read()) {
        cleanupSelectionKey(key);
        return;
    }
    // if the buffer's frame read is complete, invoke the method.
    // 如果 buffer 完全读取，则执行处理，如果失败则清理
    if (buffer.isFrameFullyRead()) {
        if (!requestInvoke(buffer)) {
            cleanupSelectionKey(key);
        }
    }
}
```

- org.apache.thrift.server.TThreadedSelectorServer#requestInvoke

处理调用，会将帧封装为 Runnable 任务，提交给线程池执行

```java
protected boolean requestInvoke(FrameBuffer frameBuffer) {
    // 封装为 Runnable
    Runnable invocation = getRunnable(frameBuffer);
    if (invoker != null) {
        try {
            // 执行处理
            invoker.execute(invocation);
            return true;
        } catch (RejectedExecutionException rx) {
            LOGGER.warn("ExecutorService rejected execution!", rx);
            return false;
        }
    } else {
        // Invoke on the caller's thread
        // 如果没有线程池，由当前线程直接处理
        invocation.run();
        return true;
    }
}
``` 

#### 处理写入事件

- org.apache.thrift.server.AbstractNonblockingServer.AbstractSelectThread#handleWrite

处理写入事件，调用 `FrameBuffer` 的写入方法进行处理

```java
protected void handleWrite(SelectionKey key) {
    FrameBuffer buffer = (FrameBuffer) key.attachment();
    if (!buffer.write()) {
        cleanupSelectionKey(key);
    }
}
```

- org.apache.thrift.server.AbstractNonblockingServer.FrameBuffer#write

由 Transport 执行写入，最终由 `SocketChannel` 执行，将响应内容发送给客户端

```java
public boolean write() {
    if (state_ == FrameBufferState.WRITING) {
        // 写入
        if (trans_.write(buffer_) < 0) {
            return false;
        }

        // 如果没有待写入的，则切换到读取
        if (buffer_.remaining() == 0) {
            prepareRead();
        }
        return true;
    }
    return false;
}
```

### 4. 执行请求

- org.apache.thrift.server.AbstractNonblockingServer.FrameBuffer#invoke

在处理读取事件时，会将 `FrameBuffer` 包装为 `Runnable`，提交给线程池执行；最终由 `FrameBuffer`处理
会获取  Processor，然后调用  process 方法进行处理

```java
public void invoke() {
    frameTrans_.reset(buffer_.array());
    response_.reset();

    try {
        // 如果有事件处理器，则触发
        if (eventHandler_ != null) {
            eventHandler_.processContext(context_, inTrans_, outTrans_);
        }
        // 获取处理器，调用处理方法
        processorFactory_.getProcessor(inTrans_).process(inProt_, outProt_);
        responseReady();
        return;
    } catch (TException te) {
    }
    // This will only be reached when there is a throwable.
    state_ = FrameBufferState.AWAITING_CLOSE;
    requestSelectInterestChange();
}
```

- org.apache.thrift.TBaseProcessor#process

在处理时，根据方法名获取具体的处理函数，然后调用响应的处理方法进行处理

```java
public void process(TProtocol in, TProtocol out) throws TException {
  TMessage msg = in.readMessageBegin();
  ProcessFunction fn = processMap.get(msg.name);
  if (fn == null) {
    TProtocolUtil.skip(in, TType.STRUCT);
    in.readMessageEnd();
    TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
    out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
    x.write(out);
    out.writeMessageEnd();
    out.getTransport().flush();
  } else {
    fn.process(msg.seqid, in, out, iface);
  }
}
```

- org.apache.thrift.ProcessFunction#process

读取请求信息，反序列化为对象，然后调用 `getResult` 方法执行实现逻辑，获取响应；如果不是 oneway 的请求，则将相应结果写入流中，发送给客户端

```java
public final void process(int seqid,
                          TProtocol iprot,
                          TProtocol oprot,
                          I iface) throws TException {

    // 获取空参数实例
    T args = getEmptyArgsInstance();
    // 读取
    args.read(iprot);
    iprot.readMessageEnd();
    TSerializable result = null;
    byte msgType = TMessageType.REPLY;
    // 获取结果
    result = getResult(iface, args);

    // 如果不是 oneway 的，则写入响应结果
    if (!isOneway()) {
        oprot.writeMessageBegin(new TMessage(getMethodName(), msgType, seqid));
        result.write(oprot);
        oprot.writeMessageEnd();
        oprot.getTransport().flush();
    }
}
```

- io.github.helloworlde.thrift.HelloService.Processor.sayHello#getResult

由生成的代码处理，会先构建一个响应结构体，然后调用相应的方法进行处理，返回结果

```java
public sayHello_result getResult(I iface, sayHello_args args) throws org.apache.thrift.TException {
  sayHello_result result = new sayHello_result();
  result.success = iface.sayHello(args.request);
  return result;
}
```

- io.github.helloworlde.thrift.HelloServiceImpl#sayHello

具体的逻辑处理，返回响应

```java
public HelloResponse sayHello(HelloMessage request) throws TException {
    String message = request.getMessage();
    HelloResponse response = new HelloResponse();
    response.setMessage("Hello " + message);
    return response;
}
```

### 4. 写入响应

- org.apache.thrift.ProcessFunction#process

在处理完请求之后，会判断是否是 oneway 请求，如果不是，则会执行写入响应

```java
if(!isOneway()) {
  oprot.writeMessageBegin(new TMessage(getMethodName(), msgType, seqid));
  result.write(oprot);
  oprot.writeMessageEnd();
  oprot.getTransport().flush();
}
```

- org.apache.thrift.protocol.TBinaryProtocol#writeMessageBegin

写入响应时，会先写入响应头；会将版本信息，消息类型，方法的名称和请求 ID 一起写入

```java
public void writeMessageBegin(TMessage message) throws TException {
  if (strictWrite_) {
    int version = VERSION_1 | message.type;
    writeI32(version);
    writeString(message.name);
    writeI32(message.seqid);
  } else {
    writeString(message.name);
    writeByte(message.type);
    writeI32(message.seqid);
  }
}
```

- io.github.helloworlde.thrift.HelloResponse.HelloResponseStandardScheme#write

随后写入响应内容，将对象序列化为字节

```java
public void write(org.apache.thrift.protocol.TProtocol oprot, HelloResponse struct) throws org.apache.thrift.TException {
  struct.validate();

  oprot.writeStructBegin(STRUCT_DESC);
  if (struct.message != null) {
    oprot.writeFieldBegin(MESSAGE_FIELD_DESC);
    oprot.writeString(struct.message);
    oprot.writeFieldEnd();
  }
  oprot.writeFieldStop();
  oprot.writeStructEnd();
}
```

然后会写入响应结尾符，由 `SelectorThread` 处理写入事件，最终将请求发送给客户端

## 参考文档

- [helloworlde/thrift-java-sample](https://github.com/helloworlde/thrift-java-sample)