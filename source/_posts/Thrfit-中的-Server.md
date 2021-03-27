---
title: Thrfit 中的 Server
date: 2021-01-18 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrfit 中的 Server 

Thrift 中有多种 Server 的实现，支持单线程、多线程、异步等多种方式

## Server 定义

### 属性

- `processorFactory_` : 处理器工厂
- `serverTransport_`: 服务端 Transport
- `eventHandler_` : 事件监听器，可以监听 Server 所有启动、关闭、处理请求相关的事件
- `inputTransportFactory_` : 输入流工厂
- `outputTransportFactory_` : 输出流工厂
- `inputProtocolFactory_` : 输入流协议工厂
- `outputProtocolFactory_` : 输出流协议工厂

### 方法

- serve

启动 Server，监听端口，对外提供服务

```java
public abstract void serve();
```

- stop 

关闭 Server，断开连接，释放并清除资源

```java
public void stop() {}
```

## 实现类

![thrift-source-server-subclass.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/thrift-source-server-subclass.png)

### 阻塞

- TSimpleServer 

Server 的简单实现，是单线程阻塞的 Server，连接实现取决于 `TServerTransport`具体类型；用于测试场景

```java
public void serve() {
    // 监听 Socket
    serverTransport_.listen();

    // 如果有事件处理器，则调用其 preSever 方法
    eventHandler_.preServe();

    // 设置运行状态
    setServing(true);

    // 只要没有关闭，就获取连接
    while (!stopped_) {
        // 接受连接
        client = serverTransport_.accept();
        connectionContext = eventHandler_.createContext(inputProtocol, outputProtocol);
        while (true) {
            // 处理上下文事件
            eventHandler_.processContext(connectionContext, inputTransport, outputTransport);
            // 处理请求
            processor.process(inputProtocol, outputProtocol);
        }
    }

    // 上下文删除事件
    eventHandler_.deleteContext(connectionContext, inputProtocol, outputProtocol);

    // 关闭 Transport
    inputTransport.close();
    outputTransport.close();
    // 修改服务状态
    setServing(false);
}
```


- TThreadPoolServer 

在 `TSimpleServer` 的基础上优化，使用了线程池处理请求；构建参数中可以指定创建线程池的参数，支持线程池饱和后超时；连接实现取决于 `TServerTransport`具体类型

```java
public void serve() {
    // 启动监听
    if (!preServe()) {
        return;
    }

    // 处理事件
    execute();
    // 等待关闭
    waitForShutdown();

    setServing(false);
}

protected boolean preServe() {
    // 监听端口
    serverTransport_.listen();
    // Run the preServe event
    // 启动事件
    eventHandler_.preServe();
    stopped_ = false;
    setServing(true);

    return true;
}

protected void execute() {
    while (!stopped_) {
        // 接受连接
        TTransport client = serverTransport_.accept();
        WorkerProcess wp = new WorkerProcess(client);
        while (true) {
            // 执行任务
            executorService_.execute(wp);
            break;
        }
    }
}
```


### 非阻塞

- AbstractNonblockingServer

`AbstractNonblockingServer` 是非阻塞的 Server 的抽象类；非阻塞 Server 有独立的线程分别处理连接和处理请求；底层实现变为 NIO，读取和写入由 `FrameBuffer` 处理

```java
// 启动 Server
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

// 处理读取
protected void handleRead(SelectionKey key) {
    // 获取帧
    FrameBuffer buffer = (FrameBuffer) key.attachment();
    // 如果没有可读取的，则清理
    if (!buffer.read()) {
        cleanupSelectionKey(key);
        return;
    }

    // 如果 buffer 完全读取，则执行处理，如果失败则清理
    if (buffer.isFrameFullyRead()) {
        if (!requestInvoke(buffer)) {
            cleanupSelectionKey(key);
        }
    }
}

// 处理写入
protected void handleWrite(SelectionKey key) {
    FrameBuffer buffer = (FrameBuffer) key.attachment();
    if (!buffer.write()) {
        cleanupSelectionKey(key);
    }
}
```

- THsHaServer

`THsHaServer` 是半同步半异步的 Server，继承自`TNonblockingServer`，是指处理连接和 IO 事件是同步的，处理请求使用线程池，是异步的；与 `TThreadPoolServer`类似，不过连接使用的是 NIO；处理连接和 IO 事件的逻辑使用 `AbstractNonblockingServer` 

```java
// 处理 IO 事件
public void run() {
    // Server 开始对外工作
    eventHandler_.preServe();
    // 只要没有停止，就执行 select 和处理连接变化
    while (!stopped_) {
        select();
        processInterestChanges();
    }
    for (SelectionKey selectionKey : selector.keys()) {
        cleanupSelectionKey(selectionKey);
    }
}

// 处理请求
protected boolean requestInvoke(FrameBuffer frameBuffer) {
    // 封装并执行调用
    Runnable invocation = getRunnable(frameBuffer);
    invoker.execute(invocation);
    return true;
}
```

- TThreadedSelectorServer

`TThreadedSelectorServer` 的性能优于 `TNonblockingServer` 和 `THsHaServer`，可以配置多个处理 IO 事件的线程，有独立的处理连接的线程，以及单独执行请求的线程池

会由 `AcceptThread` 建立连接，将连接信息添加到队列中；由 `SelectorThread` 处理 IO 事件，由线程池执行请求

```java
// 处理连接(AcceptThread)
public void run() {
    // 通知 Server 开始启动
    eventHandler_.preServe();
    while (!stopped_) {
        // 选择处理连接
        select();
    }
}

// 处理 IO 事件及连接(SelectorThread)
public void run() { 
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
}

// 处理请求
protected boolean requestInvoke(FrameBuffer frameBuffer) {
    // 封装为 Runnable
    Runnable invocation = getRunnable(frameBuffer);
    if (invoker != null) {
        // 执行处理
        invoker.execute(invocation);
        return true;
    } else {
        // Invoke on the caller's thread
        // 如果没有线程池，由当前线程直接处理
        invocation.run();
        return true;
    }
}
```