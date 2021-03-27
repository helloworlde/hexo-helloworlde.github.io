---
title: Thrfit 中的核心概念
date: 2021-01-17 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrfit 中的核心概念

## 服务端

```
  +-------------------------------------------+
  | Server                                    |
  | (single-threaded, event-driven etc)       |
  +-------------------------------------------+
  | Processor                                 |
  | (compiler generated)                      |
  +-------------------------------------------+
  | Protocol                                  |
  | (JSON, compact etc)                       |
  +-------------------------------------------+
  | Transport                                 |
  | (raw TCP, HTTP etc)                       |
  +-------------------------------------------+
```

Thrift Server 设计大致可以分为四层，分别是：

- Server：负责连接调度、服务的生命周期，定义接口是`TServer`
		- `TSimpleServer`：简单的阻塞服务端
		- `TThreadPoolServer`：使用线程池的处理请求的阻塞服务端
		- `THsHaServer`：使用线程池处理请求的基于 NIO 的非阻塞服务端
		- `TThreadedSelectorServer`：使用多种线程池的基于 NIO 的非阻塞服务端

- Processor：处理请求，具体的实现由生成的代码处理，定义接口是 `TProcessor`
	- `TBaseProcessor`：同步处理的 Processor
	- `TBaseAsyncProcessor`：异步处理的 Processor
	- `TMultiplexedProcessor`：支持多个服务的同步 Processor

- Protocol：请求协议，数据的编解码实现，定义接口是 `TProtocol`
	- `TBinaryProtocol`：二进制协议
	- `TCompactProtocol`：压缩协议
	- `TJSONProtocol`：JSON 格式协议
	- `TMultiplexedProtocol`：支持多个 Processor 的封装协议，依赖于其他协议

- Transport：底层的连接，提供了读写的抽象实现；服务端定义是 `TServerTransport`
	- `TServerSocket`: 基于 `ServerSocket` 的服务端 Transport
	- `TNonblockingServerSocket`：基于 `ServerSocketChannel` 的服务端 Transport
	- `TSaslTransport`：支持 SSL 加密的 Transport

## 客户端

```
  +-------------------------------------------+
  | Client                                    |
  | (sync or async generated)                 |
  +-------------------------------------------+
  | Protocol                                  |
  | (JSON, compact etc)                       |
  +-------------------------------------------+
  | Transport                                 |
  | (raw TCP, HTTP etc)                       |
  +-------------------------------------------+
```

Thrift Client 设计分为三层，分别是：

- Client：客户端，用于发起请求，接收响应
		- `TServiceClient`：同步调用的客户端
		- `TAsyncClient`：异步调用的客户端

- Protocol：请求协议，数据的编解码实现，定义接口是 `TProtocol`
	- `TBinaryProtocol`：二进制协议
	- `TCompactProtocol`：压缩协议
	- `TJSONProtocol`：JSON 格式协议
	- `TMultiplexedProtocol`：支持多个 Processor 的封装协议，依赖于其他协议

- Transport：底层的连接，提供了读写的抽象实现；客户端的定义是 `TTransport`
	- `TFileTransport`: 读写文件的 Tranport
	- `THttpClient`：基于 `HttpClient` 的 Transport
	- `TNonblockingSocket`：基于 `SocketChannel`的 Transport
	- `TSocket`: 基于 `Socket` 的 Tranport
	- `TSaslClientTransport`: 支持 SASL 鉴权认证的 Tranport
	- `TZlibTransport`：支持压缩的 Transport
	- `TSaslTransport`：支持 SSL 加密的 Transport



## 参考文档 

- [Concepts](https://thrift.apache.org/docs/concepts.html)