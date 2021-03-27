---
title: gRPC Channel
date: 2020-11-18 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC Channel

`Channel` 是用于执行 RPC 请求的概念上的端点连接，基于负载和配置，一个 `Channel` 可以有 0 或多个真实连接

`Subchannel` 代表逻辑连接，最多维护一个物理连接发送 RPC，对应多个 Transport 

![grpc-source-code-channel-class-diagram.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-channel-class-diagram.png)

Channel 有多个子类：
- `ManagedChannel`： 实现了生命周期管理能力的抽象子类
	- `ManagedChannelImpl`： `ManagedChannel` 的实现类，`Channel` 的主要实现
	- `ManagedChannelOrphanWrapper`: `ManagedChannel` 的包装类，用于引用 `ManagedChannel`
- `RealChannel`：真正执行创建 `ClientCallImpl` 实例 
- `SubchannelAsChannel`: 将 `Subchannel` 转为 `Channel`

Subchannel 的子类：
- `SubchannelImpl`：Subchannel 的实现类
 
## 方法

### Channel 

- 发起调用

```java
public abstract <RequestT, ResponseT> ClientCall<RequestT, ResponseT> newCall(MethodDescriptor<RequestT, ResponseT> methodDescriptor, CallOptions callOptions);
```

- 目标地址 

```java
public abstract String authority();
```

### ManagedChannel 

`ManagedChannel` 是 `Channel` 的子类，提供生命周期管理的 `Channel`；由 `ManagedChannelImpl` 实现功能

##### 关闭

- shutdown 

初始化一个顺序关闭，既有的调用会继续执行，但是新的调用会被立即取消

```java
public abstract ManagedChannel shutdown();
```

- shutdownNow

强制关闭 Channel，会取消所有的调用；即使是强制关闭也不会瞬间停止

```java
public abstract ManagedChannel shutdownNow();
```


- isShutdown 

返回 `Channel` 是否终止，终止的 `Channel` 没有执行中的调用，相关的资源被释放

```java 
public abstract boolean isShutdown();
``` 

- awiatTermination 

等待 `Channel` 变为终止，如果超时则放弃等待

```java
public abstract boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
```


#### 状态

- getState

获取当前连接的状态，当参数为 true 是，如果 `Channel` 处于  `IDLE` 模式，则会退出 `IDLE` 模式，尝试建立连接

```java
public ConnectivityState getState(boolean requestConnection) {
        throw new UnsupportedOperationException("Not implemented");
}
```

- notifyWhenStateChanged

当 `Channel` 的状态和给定的状态不同时触发，用于执行状态变化的回调

```java
public void notifyWhenStateChanged(ConnectivityState source, Runnable callback) {
        throw new UnsupportedOperationException("Not implemented");
}
```

- enterIdle

进入空闲模式，`Channel` 的状态将变为 `IDLE`，触发 `Channel` 的名称解析和负载均衡关闭，`Channel` 上执行中的请求会继续执行，新的 RPC 请求会触发建立新的连接

```java
public void enterIdle() {
}
```

- resetConnectBackoff

对于 `TRANSIENT_FAILURE` 状态的 `Subchannel`，使退避计时器短路，并立即重新连接，也会尝试 `NameResovler#refresh`；主要用于 Android 平台

```java
public void resetConnectBackoff() {
}
```

### Subchannel

#### 生命周期

- start 

开始 `Subchannel`，只能调用一次，同时会启动监听器

```java
public void start(SubchannelStateListener listener) {
      throw new UnsupportedOperationException("Not implemented");
}
```

- requestConnection

如果没有活跃的连接，在要求建立活跃的连接

```java
public abstract void requestConnection()
```

- shutdown 

关闭 `Subchannel`，当调用这个方法后，`SubchannelPicker` 不会再返回这个 `Subchannel`

```java
public abstract void shutdown();
```

#### 操作地址

- getAddresses 

获取第一个地址集合

```java
public final EquivalentAddressGroup getAddresses() {
  List<EquivalentAddressGroup> groups = getAllAddresses();
  Preconditions.checkState(groups.size() == 1, "%s does not have exactly one group", groups);
  return groups.get(0);
}
```

- getAllAddresses

返回 `Subchannel` 绑定的所有地址

```java
public List<EquivalentAddressGroup> getAllAddresses() {
      throw new UnsupportedOperationException();
}
```

- updateAddresses

更新 `Subchannel` 的地址

```java
public void updateAddresses(List<EquivalentAddressGroup> addrs) {
      throw new UnsupportedOperationException();
}
```

#### 其他方法 

- asChannel 

使用当前的 `Subchannel` 创建 `Channel`，用于健康检查等内部操作

```java
public Channel asChannel() {
      throw new UnsupportedOperationException();
}
```

- getInternalSubchannel

获取 `Subchannel` 的内部的 `Subchannel`，表示用于发送 RPC 的基础的 `Subchannel`

```java
public Object getInternalSubchannel() {
      throw new UnsupportedOperationException();
}
```

- getAttributes

获取 `Subchannel 的属性

```java
public abstract Attributes getAttributes();
```