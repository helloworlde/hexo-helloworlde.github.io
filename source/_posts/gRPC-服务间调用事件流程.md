---
title: gRPC 服务间调用事件流程
date: 2021-02-20 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 服务间调用事件流程

## 调用流程图

![gRPC请求事件流程.svg](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/gRPC请求事件流程.svg)

## 可监听的事件

### 客户端

#### ClientCall

客户端调用，用于执行客户端的调用行为

- `checkStart`：开始调用
- `request`：指定发送消息的数量
- `sendMessage`：发送消息到缓冲区
- `halfClose`：半关闭，会将消息发送给 Server 端
- `cancel`：调用失败时取消

#### ClientCall.Listener

调用监听器，监听调用事件

-  `onReady`：流就绪事件，用于非 `UNARY` 和 `SERVER_STREAM` 的请求
-  `onHeaders`：当接收到 Server 端返回的 Header 时调用
-  `onMessage`：当接收到 Server 端返回的 Message 时调用
-  `onClose`：当流关闭时调用

#### ClientStreamTracer

流统计追踪，监听流的事件

- `outboundHeaders`：发送 header 给 Server 端
- `outboundMessage`：发送 message 给 Server 端
- `inboundHeaders`：接收 Server 端返回的 headers
- `inboundMessage`：接收 Server 端返回的 message
- `inboundTrailers`：接收 Server 端返回的 trailers
- `streamClosed`：流关闭时调用

### 服务端

#### ServerTransportFilter

Server 端 Transport 事件过滤器，支持监听事件，修改 Transport 的属性

- `transportReady`：Transport 就绪事件
- `transportTerminated`：Transport 关闭事件

#### ServerStreamTracer

Server 端流事件追踪，监听流的事件

- `filterContext`：创建 Context 时调用，支持修改 Context 的属性
- `serverCallStarted`：创建 ServerCall 时调用
- `inboundMessage`：接收客户端发送的 message
- `outboundMessage`：发送 message 给客户端
- `streamClosed`：流关闭时调用

#### ServerCall

Server 端调用，执行 Server 端调用行为

- `request`：要求指定数量的消息
- `sendHeaders`：发送 headers 给客户端
- `sendMessage`：发送 message 给客户端
- `close`：调用完成时关闭

#### ServerCall.Listener

Server 端调用监听器

- `onReady`：流就绪事件
- `onMessage`：接收到消息
- `onHalfClose`：接收到半关闭事件
- `onCancel`：流取消事件
- `onComplete`：流完成事件


