---
title: Thrift 中的 Transport
date: 2021-02-01 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrift 中的 Transport 

Thrift 中有 `TTransport` 和 `TServerTransport`，封装了底层传输层的数据读写；分别用于客户端和服务端

## TTransport

![thrift-java-source-class-transport.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/thrift-java-source-class-transport.png)

### 方法 

- open 

用于建立与 Server 端的连接

```java
public abstract void open() throws TTransportException;
```

- close

关闭连接

```java
public abstract void close();
```

- read

用于读取数据

```java
public abstract int read(byte[] buf, int off, int len) throws TTransportException;
```

- write

用于写入数据

```java
public abstract void write(byte[] buf, int off, int len) throws TTransportException;
```

- flush

清空缓冲区中的数据，发送给服务端

```java
public void flush() throws TTransportException {
}
```

### 实现类

#### 非封装的 Transport 

- `TNonblockingTransport`: 非阻塞的 Transport 的抽象类，底层使用 NIO 
- `TNonblockingSocket`: `TNonblockingTransport` 的实现类，基于 SocketChannel 的 Transport，是非阻塞的
- `TIOStreamTransport`: 基于 IO 流的 Transport 
- `TSocket`: `TIOStreamTransport` 的子类，底层使用 `Socket`
- `TSimpleFileTransport`：基于文件的 Transport，会将流写入文件或者从文件读取流
- `TFileTransport`: 基于文件的 Transport，会将流写入文件或者从文件读取流
- `THttpClient`：基于 `HttpClient` 或 `HttpURLConnection`，会通过 HTTP 的方式发送请求，通常用于 `TServlet` 的服务端
- `ByteBuffer`: 基于 ByteBuffer 的 Transport
- `TMemoryInputTransport`：基于内存数组的 Transport，会从底层的数组读取，用于测试场景
- `TMemoryBuffer`：使用内存数组作为缓冲区的 Transport，用于测试场景

#### 封装的 Transport

- `TZlibTransport`: 压缩的 Transport，会将流压缩后再发送
- `AutoExpandingBufferReadTransport`: 可扩展读缓冲区的 Transport，使用可变数组作为缓冲区
- `AutoExpandingBufferWriteTransport`: 可扩展写缓冲区的 Transport，使用可变数组作为缓冲区
- `TSaslTransport`：支持 SASL(Simple Authentication and Security Layer) 认证的 Transport，有两个实现类，用于客户端的`TSaslClientTransport` 和用于服务端的 `TSaslServerTransport`
- `TFramedTransport`：缓冲的 Transport，通过在前面带有4字节帧大小的消息来确保每次都完全读取消息
- `TFastFramedTransport`： 复用并扩展了读写缓冲区的 Transport，避免每次都创建新的 byte 数组

## TServerTransport

![thrift-java-source-class-server-transport.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/thrift-java-source-class-server-transport.png)

### 方法

- listen

监听指定的端口

```java
public abstract void listen() throws TTransportException;
```

- accept

用与接受连接

```java
public final TTransport accept() throws TTransportException {
    TTransport transport = acceptImpl();
    if (transport == null) {
        throw new TTransportException("accept() may not return NULL");
    }
    return transport;
}
```

- close

断开连接，停止监听端口，关闭服务

```java
public abstract void close();
```

### 实现类

- `TNonblockingServerTransport`：非阻塞服务端抽象类，提供了选择器的注册
- `TNonblockingServerSocket`： `TNonblockingServerTransport`的实现类，底层使用 NIO 的 `ServerSocketChannel`非阻塞 Transport
- `TServerSocket`：使用 `ServerSocket` 的阻塞 Transport