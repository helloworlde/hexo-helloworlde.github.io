---
title: gRPC 命名解析
date: 2020-09-20 22:35:22
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 命名解析

命名解析根据服务的 URI，从注册中心获取并解析服务实例 IP，默认支持 schema 为 DNS，grpclb，xds 等

![grpc-source-code-name-resolver-diagram.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-name-resolver-diagram.png)

gRPC 的命名解析的父类接口是 `NameResolver`
![grpc-source-code-name-resolver-class.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-name-resolver-class.png)

`NameResolver` 包含有多个子类，用于实现命名解析
每个 `NameResolver` 都有一个 `Provider`，用于创建 `NameResolver` 实例；所有的 `Provider` 都注册到 `NameResolverRegistry` 中，`NameResolverRegistry` 创建 `Factory` 实例，最终通过 `Provider` 创建 `NameResolver`

![grpc-source-code-name-resolver-with-sub-class.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-name-resolver-with-sub-class.png)

命名解析的整个工作流程是：
1. 使用 `NameResolverRegistry` 或者 SPI 方式注册 Provider
2. 调用 Channel 的 `build` 方法创建 `NameResovler.Factory`
3. 根据 Factory 最终调用 Provider 创建 `NameResolver`，
2. 创建 `Listener` 的实例
3. 调用 `NameResolver` 的 `start` 方法，传入 `Listener` 实例
4. 创建 `Runnable` 任务，通过调用 `Listener` 的 `onResult` 方法进行更新

## 创建 NameResolver 

在 Channel 调用 `build` 方式时，会在 `io.grpc.internal.ManagedChannelImpl#ManagedChannelImpl`的构造方法中获取 `NameResolver.Factory`，这个属性的值是由调用 `io.grpc.internal.AbstractManagedChannelImplBuilder#getNameResolverFactory` 方法获取的，这个方法里面的属性值来自于 `io.grpc.NameResolverRegistry#asFactory`,`NameResolverRegistry` 自己通过内部类 `NameResolverFactory`创建了`NameResovler.Factory` 的实例，在`io.grpc.internal.ManagedChannelImpl#getNameResolver`中调用 Factory 的 `newNameResolver`时，从 `provider` 属性中获取根据优先级排序后的 Provider，通过 Provider 创建 `NameResolver` 实例并返回第一个有效实例

1. 创建 Channel 时注册 `NameResovlerProvider`

```java
NameResolverRegistry.getDefaultRegistry().register(new DnsNameResolverProvider());
```

2. 在 Channel 中获取 `Factory`

```java
// 服务发现工厂
this.nameResolverFactory = builder.getNameResolverFactory();
```

3. 在 `NameResolverRegistry` 中初始化 `Factory`，构造 `NameResolverFactory` 实例，在 `asFactory` 方法中返回

```java
private final NameResolver.Factory factory = new NameResolverFactory();

// 返回该实例
private NameResolver.Factory nameResolverFactory = nameResolverRegistry.asFactory();
```

4. 获取 `NameResovler` 实例

创建 `NameResolver` 的 `Factory` 是通过 Channel 的 `Builder` 传入的，`Args` 是在 Channel 的构造方法中创建的

- io.grpc.internal.ManagedChannelImpl#ManagedChannelImpl

```java
    // 命名解析器参数
    this.nameResolverArgs = NameResolver.Args.newBuilder()
                                             .setDefaultPort(builder.getDefaultPort())
                                             .setProxyDetector(proxyDetector)
                                             .setSynchronizationContext(syncContext)
                                             .setScheduledExecutorService(scheduledExecutor)
                                             .setServiceConfigParser(serviceConfigParser)
                                             .setChannelLogger(channelLogger)
                                             .setOffloadExecutor(
                                                     // Avoid creating the offloadExecutor until it is first used
                                                     new Executor() {
                                                       @Override
                                                       public void execute(Runnable command) {
                                                         offloadExecutorHolder.getExecutor().execute(command);
                                                       }
                                                     })
                                             .build();

    // 命名解析
    this.nameResolver = getNameResolver(target, nameResolverFactory, nameResolverArgs);
```

- io.grpc.internal.ManagedChannelImpl#getNameResolver

根据地址，创建 `NameResolver` 实例，如果 URI 缺少 `Schema`，则添加默认的 `Schema`

```java
  static NameResolver getNameResolver(String target,
                                      NameResolver.Factory nameResolverFactory,
                                      NameResolver.Args nameResolverArgs) {
    // 解析地址
    URI targetUri = new URI(target);

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

- io.grpc.internal.DnsNameResolverProvider#newNameResolver

在 NameResolverProvider 中创建 NameResolver，调用具体实现类的构造方法，初始化相应的参数

```java
    public DnsNameResolver newNameResolver(URI targetUri, NameResolver.Args args) {
        // 如果是 DNS 开头的 Schema
        if (SCHEME.equals(targetUri.getScheme())) {
            // 检查要解析的服务不为空，且以斜线开头
            String targetPath = Preconditions.checkNotNull(targetUri.getPath(), "targetPath");
            Preconditions.checkArgument(targetPath.startsWith("/"), "the path component (%s) of the target (%s) must start with '/'", targetPath, targetUri);

            // 截取斜线之后的部分作为服务名
            String name = targetPath.substring(1);
            // 创建 DNS 服务解析器
            return new DnsNameResolver(
                    targetUri.getAuthority(),
                    name,
                    args,
                    GrpcUtil.SHARED_CHANNEL_EXECUTOR,
                    Stopwatch.createUnstarted(),
                    InternalServiceProviders.isAndroid(getClass().getClassLoader()));
        } else {
            // 如果不是 DNS 开头的则返回 null
            return null;
        }
    }

```

## Listener

- io.grpc.internal.ManagedChannelImpl#exitIdleMode

创建负载均衡，和服务解析实例作为参数，创建 `Listener`

```java
  void exitIdleMode() {
    LbHelperImpl lbHelper = new LbHelperImpl();
    // 自动配置负载均衡
    lbHelper.lb = loadBalancerFactory.newLoadBalancer(lbHelper);
    this.lbHelper = lbHelper;
    // 服务发现监听器
    NameResolverListener listener = new NameResolverListener(lbHelper, nameResolver);
    nameResolver.start(listener);
  }
```

- io.grpc.internal.DnsNameResolver#start

`start` 方法最终通过线程池执行 `Resolve` 任务

```java
    @Override
    public void start(Listener2 listener) {
        if (usingExecutorResource) {
            executor = SharedResourceHolder.get(executorResource);
        }
        // 解析
        resolve();
    }

    private void resolve() {
        if (resolving || shutdown || !cacheRefreshRequired()) {
            return;
        }
        resolving = true;
        // 根据监听器，解析名称
        executor.execute(new Resolve(listener));
    }
```

## 解析地址

- io.grpc.internal.DnsNameResolver.Resolve

`Resolve` 实现了 `Runnable` 接口，在 `run` 方法中，先调用 `doResolve` 方法，将目标 URI 解析为地址集合，同时也会获取配置
然后根据获取的地址和配置为 `ResolutionResult.Builder`  赋值，调用 `Listener2` 的 `onResult` 方法处理更新的结果

```java
    private final class Resolve implements Runnable {
        private final Listener2 savedListener;

        /**
         * 从 DNS 解析服务
         */
        @Override
        public void run() {
            InternalResolutionResult result = null;
            try {
                ResolutionResult.Builder resolutionResultBuilder = ResolutionResult.newBuilder();
                // 代理的地址
                // 根据 HOST 解析地址和配置
                result = doResolve(false);

                if (result.error != null) {
                    savedListener.onError(result.error);
                    return;
                }
                // 如果有地址，则设置地址
                if (result.addresses != null) {
                    resolutionResultBuilder.setAddresses(result.addresses);
                }
                if (result.config != null) {
                    resolutionResultBuilder.setServiceConfig(result.config);
                }
                if (result.attributes != null) {
                    resolutionResultBuilder.setAttributes(result.attributes);
                }
                // 更新负载均衡策略，处理未处理的请求
                savedListener.onResult(resolutionResultBuilder.build());
            } catch (IOException e) {
                savedListener.onError(Status.UNAVAILABLE.withDescription("Unable to resolve host " + host).withCause(e));
            }
        }
    }
```

## 更新实例

- io.grpc.internal.ManagedChannelImpl.NameResolverListener#onResult

`NameResolverListener` 是 `Listener2` 的实现类；`Listener2` 是 `Listener` 接口的抽象实现，新增了一个 `onResult` 方法

在 `onResult` 方法中有个 `Runnable`，`run` 方法会根据传入的结果，更新服务的配置；然后根据实例地址，更新负载均衡的实例列表

任务由 `SynchronizationContext` 执行

```java
    @Override
    public void onResult(final ResolutionResult resolutionResult) {
      final class NamesResolved implements Runnable {
        @Override
        public void run() {
          List<EquivalentAddressGroup> servers = resolutionResult.getAddresses();
          // 更新传入的配置 ...
          
          ManagedChannelServiceConfig effectiveServiceConfig;
          handleServiceConfigUpdate();

          // 获取属性
          Attributes effectiveAttrs = resolutionResult.getAttributes();
          // 如果服务发现没有关闭
          if (NameResolverListener.this.helper == ManagedChannelImpl.this.lbHelper) {

            // 更新负载均衡实例
            Status handleResult = helper.lb.tryHandleResolvedAddresses(
                    ResolvedAddresses.newBuilder()
                                     .setAddresses(servers)
                                     .setAttributes(effectiveAttrs)
                                     .setLoadBalancingPolicyConfig(effectiveServiceConfig.getLoadBalancingConfig())
                                     .build());
          }
        }
      }

      // 执行处理
      syncContext.execute(new NamesResolved());
    }
```
