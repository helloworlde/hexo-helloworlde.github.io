---
title: gRPC 中的核心概念
date: 2020-09-20 22:40:59
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 中的核心概念

## Stub

Stub 层暴露给开发者，提供类型安全的绑定到正在适应的数据模型/IDL/接口

## Channel 

Channel 层是传输处理之上的抽象，适合拦截器、装饰器，并比 Stub 层暴露更多的行为给应用
一个 Channel 可能有多个 Subchannel 

### Subchannel 
Subchannel 代表负载均衡过的 Channel

### Channel 状态

- CONNECTING
Channel 正在尝试建立连接，并且正在等待名称解析，TCP连接建立或TLS握手所涉及的步骤之一，创建时可以将其用作通道的初始状态

- READY
Channel 握手成功建立连接，并且所有的后续通信尝试均已成功(或未发生任何失败)

- TRANSIENT_FAILURE
发生了一些瞬时故障（例如TCP 3握手超时或套接字错误），处于此状态的通道最终将切换到 CONNECTING 状态，并尝试再次建立连接。由于重试是通过指数退避完成的，因此无法连接的通道在此状态下将开始花费很少的时间，但是由于尝试反复失败，因此通道在此状态下将花费越来越多的时间。对于许多非致命故障（例如，由于服务器尚不可用，TCP连接尝试超时），通道可能会在此状态下花费越来越多的时间

- IDLE
由于缺少新的或待处理的RPC，Channel 甚至没有尝试创建连接的状态；可以在这种状态下创建新的RPC，任何在通道上启动RPC的尝试都将使该 Channel 退出此状态以进行连接。如果指定的 IDLE_TIMEOUT 的通道上没有 RPC 活动，即在此期间没有新的或挂起的（活动的）RPC，则 READY 或 CONNECTING 的通道将切换到 IDLE；此外，在没有活动或暂挂RPC的情况下接收 GOAWAY 的通道也应切换到 IDLE，以避免试图断开连接的服务器上的连接过载。我们将使用默认的IDLE_TIMEOUT 300秒（5分钟）。
 
 - SHUTDOWN 
此频道已开始关闭，任何新的RPC应该立即失败，待处理的RPC可能会继续运行，直到应用程序将其取消为止。通道可能进入此状态，原因是应用程序明确请求关闭，或者尝试进行连接通信期间发生了不可恢复的错误。（截至2015年6月12日，（连接或通讯时没有已知的错误被归类为不可恢复。）进入此状态的通道永远不会离开此状态。

##  Transport 

Transport 层承担在线上放置和获取字节的工作，被建模为 Stream 工厂，是真正的连接

gRPC 有三个 Transport 实现：
1. 基于 Netty的 Transport 是主要的  Transport 实现，基于 Netty，可同时用于客户端和服务端
2. 基于 OkHttp 的 Transport 是轻量级的 Transport，基于 OkHttp，主要用于 Android 端并只作为客户端
3. InProcess Transport 是当服务器和客户端在同一个进程内时使用，用于测试
