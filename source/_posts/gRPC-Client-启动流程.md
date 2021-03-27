---
title: gRPC Client 启动流程
date: 2020-11-17 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC Client 启动流程

gRPC 启动初始化的流程，使用 Netty 作为底层的实现

##  初始化 Channel 

Channel 的初始化通过 `ChannelBuilder` 构建
这里通过 `forTarget` 设置了要解析的服务名称，会通过 `NameResolver` 解析，转换为具体的地址

```java
ManagedChannel channel = ManagedChannelBuilder.forTarget("grpc-server")
                                              .usePlaintext()
                                              .build();
``` 

- 构建 ManagedChannel 实例

`io.grpc.internal.AbstractManagedChannelImplBuilder#build`

调用 build 时，会根据 builder 中的属性，创建 `ManagedChannelImpl` 的实例 

```java
public ManagedChannel build() {
    return new ManagedChannelOrphanWrapper(new ManagedChannelImpl(
            this,
            // 构建 Transport 工厂
            buildTransportFactory(),
            new ExponentialBackoffPolicy.Provider(),
            // 线程池
            SharedResourcePool.forResource(GrpcUtil.SHARED_CHANNEL_EXECUTOR),
            // 计时器
            GrpcUtil.STOPWATCH_SUPPLIER,
            // 统计和追踪拦截器
            getEffectiveInterceptors(),
            // 时间提供器
            TimeProvider.SYSTEM_TIME_PROVIDER));
}
```


## 初始化 Channel 属性

- io.grpc.internal.ManagedChannelImpl#ManagedChannelImpl

为 `ManagedChannel` 设置属性，初始化服务发现，负载均衡，拦截器等并创建真正的 Channel 


```java
ManagedChannelImpl(AbstractManagedChannelImplBuilder<?> builder,
        ClientTransportFactory clientTransportFactory,
        BackoffPolicy.Provider backoffPolicyProvider,
        ObjectPool<? extends Executor> balancerRpcExecutorPool,
        Supplier<Stopwatch> stopwatchSupplier,
        List<ClientInterceptor> interceptors,
        final TimeProvider timeProvider)
```

##### 设置服务名称

```java
this.target = checkNotNull(builder.target, "target");
```

##### 设置  TransportFactory

创建了支持鉴权的代理的 `TransportFactory`，用于支持向服务端发起请求进行鉴权

```java
this.transportFactory = new CallCredentialsApplyingTransportFactory(clientTransportFactory, this.executor);
```


##### 构建服务发现工厂

```java
this.nameResolverFactory = builder.getNameResolverFactory();
```

- io.grpc.internal.AbstractManagedChannelImplBuilder#getNameResolverFactory

如果没有覆盖服务名称，则使用这个 `nameResolverFactory`，否则使用 `OverrideAuthorityNameResolverFactory`

```java
NameResolver.Factory getNameResolverFactory() {
        if (authorityOverride == null) {
            return nameResolverFactory;
        } else {
            return new OverrideAuthorityNameResolverFactory(nameResolverFactory, authorityOverride);
        }
    }
```

##### 构建服务发现实例

```java
this.nameResolver = getNameResolver(target, nameResolverFactory, nameResolverArgs);
``` 

- io.grpc.internal.ManagedChannelImpl#getNameResolver

```java
static NameResolver getNameResolver(String target,
                                    NameResolver.Factory nameResolverFactory,
                                    NameResolver.Args nameResolverArgs) {
  URI targetUri = null;
  StringBuilder uriSyntaxErrors = new StringBuilder();
  // 解析地址
  targetUri = new URI(target);

  // 创建 NameResolver
  if (targetUri != null) {
    NameResolver resolver = nameResolverFactory.newNameResolver(targetUri, nameResolverArgs);
    if (resolver != null) {
      return resolver;
    }
  }

  // 如果不是 URI 格式，则使用默认的 schema
  if (!URI_PATTERN.matcher(target).matches()) {
    targetUri = new URI(nameResolverFactory.getDefaultScheme(), "", "/" + target, null);

    NameResolver resolver = nameResolverFactory.newNameResolver(targetUri, nameResolverArgs);
    if (resolver != null) {
      return resolver;
    }
  }
  throw new IllegalArgumentException(String.format("cannot find a NameResolver for %s%s", target, uriSyntaxErrors.length() > 0 ? " (" + uriSyntaxErrors + ")" : ""));
}
```


##### 设置是否开启重试

根据是否主动开启了重试和禁止重试的开关决定是否要重试

```java
this.retryEnabled = builder.retryEnabled && !builder.temporarilyDisableRetry;
```

##### 设置负载均衡

根据负载均衡策略构建 `LoadBalancerFactory`，会从注册器中获取并初始化负载均衡实例

```java
this.loadBalancerFactory = new AutoConfiguredLoadBalancerFactory(builder.defaultLbPolicy);
```


##### 配置信息

```java
// 配置解析器
ScParser serviceConfigParser = new ScParser(
        retryEnabled,
        builder.maxRetryAttempts,
        builder.maxHedgedAttempts,
        loadBalancerFactory,
        channelLogger);

// 服务配置拦截器
serviceConfigInterceptor = new ServiceConfigInterceptor(retryEnabled);

// 如果 builder 有配置，则解析配置
if (builder.defaultServiceConfig != null) {
  // 解析配置
  ConfigOrError parsedDefaultServiceConfig = serviceConfigParser.parseServiceConfig(builder.defaultServiceConfig);
  this.defaultServiceConfig = (ManagedChannelServiceConfig) parsedDefaultServiceConfig.getConfig();
  this.lastServiceConfig = this.defaultServiceConfig;
} else {
  this.defaultServiceConfig = null;
}

this.lookUpServiceConfig = builder.lookUpServiceConfig;

// 如果没有开启则使用默认配置
if (!lookUpServiceConfig) {
  if (defaultServiceConfig != null) {
    channelLogger.log(ChannelLogLevel.INFO, "Service config look-up disabled, using default service config");
  }
  handleServiceConfigUpdate();
}
```

##### 创建 RealChannel 

`RealChannel` 用于发起请求

```java
Channel channel = new RealChannel(nameResolver.getServiceAuthority());
```

- io.grpc.internal.ManagedChannelImpl.RealChannel#newCall

```java
public <ReqT, RespT> ClientCall<ReqT, RespT> newCall(MethodDescriptor<ReqT, RespT> method,
                                                     CallOptions callOptions) {
  return new ClientCallImpl<>(method,
          // 执行的线程池
          getCallExecutor(callOptions),
          // 调用的参数
          callOptions,
          // Transport 提供器
          transportProvider,
          // 如果没有关闭，则获取用于调度的执行器
          terminated ? null : transportFactory.getScheduledExecutorService(),
          // 统计 Channel 调用信息
          channelCallTracer,
          // 是否重试
          retryEnabled)
          .setFullStreamDecompression(fullStreamDecompression)
          .setDecompressorRegistry(decompressorRegistry)
          .setCompressorRegistry(compressorRegistry);
}
```

##### 拦截器

```java
// 添加方法拦截器
channel = ClientInterceptors.intercept(channel, serviceConfigInterceptor);
```

除此之外，还初始化了一些其他的 Channel 的配置和属性，当构建 Channel 完成后，就可以使用 Channel 构建 Stub，用于发起请求