---
title: gRPC  负载均衡
date: 2020-09-20 22:36:58
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC  负载均衡

gRPC 内定义了  LoadBalancer 接口，用于负载均衡

![grpc-source-code-loadbalancer-methods.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-loadbalancer-methods.png)

LoadBalancer 中的主要方法
- `handleResolvedAddress`：处理  `NameResolver` 解析的地址，用于创建 `Subchannel`
- `handleNameResolutionError`: 处理命名解析失败，会销毁已经存在的 `Suchannel`
- `requestConnection`: 创建连接，会为 `Subchannel` 初始化 `Transport`，并建立连接

![grpc-source-code-loadbalancer-sub-class.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-loadbalancer-sub-class.png)

LoadBalancer 接口有多个实现类，如用于代理的 `ForwardingLoadBalancer`；基于策略的 `RoundRobinLoadBalancer`,`PickFirstLoadBalancer`, `GrpclbLoadBalancer`等；支持扩展功能的`HealthCheckingLoadBalancer`, `GracefulSwitchLoadBalancer` 等

![grpc-source-code-loadbalancer-class-diagram.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grpc-source-code-loadbalancer-class-diagram.png)

LoadBalancer 有多个内部类，用于实现负载均衡

- `Factory`: 用于创建 `LoadBalancer`，通过  `LoadBalancerProvider` 实现
- `Subchannel`: 逻辑连接，一个 `Subchannel` 内可能包含多个 `IP:PORT`
- `Helper`: 用于创建 `LoadBalancer`、`Subchannel` 等
- `SubchannelPicker`: `Subchannel` 选择器，根据不同的策略使用不同的选择方式
- `SubchannelStateListener`: `Subchannel` 状态监听器，当 `Subchannel` 状态发生变化时及时更新

LoadBalancer 的工作流程是：
1. 使用 `LoadBalancerRegistry` 或者 SPI 的方式注册 `LoadBalancerProvider`
2. 调用 Channel Builder 的 `defaultLoadBalancingPolicy` 设置负载均衡策略
3. 在 `ManagedChannelImpl` 的构造方法中，创建 `Factory`
4. 在 `ManagedChannelImpl#exitIdleMode` 中创建 `LoadBalancer` 实例
5. 将创建的实例作为参数传递给 `NameResolverListener`
6. 当 `NameResolver` 解析服务名称后，最终调用 `handleResolvedAddresses `方法，根据不同的策略进行处理
7. `LoadBalancer` 根据解析的地址创建 `Subchannel`
8. `Subchannel`调用 `requestConnection` 方法建立连接

## 创建 LoadBalancer 

1. 创建 Channel 前注册 Provider 

```java
LoadBalancerRegistry.getDefaultRegistry().register(new HealthCheckingRoundRobinLoadBalancerProvider());
```

2. 创建 Channel 时设置负载均衡策略

```java
ManagedChannelBuilder.forTarget("server")
                     .defaultLoadBalancingPolicy("round_robin")
                     .build();
```

3. 在 `io.grpc.internal.ManagedChannelImpl#ManagedChannelImpl` 构造方法中初始化 Factory

Factory 的实现类是 `AutoConfiguredLoadBalancerFactory`

```java
this.loadBalancerFactory = new AutoConfiguredLoadBalancerFactory(builder.defaultLbPolicy);
```

4. 在 `io.grpc.internal.ManagedChannelImpl#exitIdleMode`时创建 `LoadBalancer` 实例

```java
// 构建新的 lbHelper
LbHelperImpl lbHelper = new LbHelperImpl();

// 自动配置负载均衡
lbHelper.lb = loadBalancerFactory.newLoadBalancer(lbHelper);

this.lbHelper = lbHelper;
```

5. 创建 `LoadBalancer` 实例

- `AutoConfiguredLoadBalancer`

```java
    public AutoConfiguredLoadBalancer newLoadBalancer(Helper helper) {
      return new AutoConfiguredLoadBalancer(helper);
    }

    AutoConfiguredLoadBalancer(Helper helper) {
      this.helper = helper;
      // 从注册器中获取默认的负载均衡策略提供器
      delegateProvider = registry.getProvider(defaultPolicy);
      if (delegateProvider == null) {
        throw new IllegalStateException("Could not find policy '" + defaultPolicy
                + "'. Make sure its implementation is either registered to LoadBalancerRegistry or"
                + " included in META-INF/services/io.grpc.LoadBalancerProvider from your jar files.");
      }
      // 创建新的
      delegate = delegateProvider.newLoadBalancer(helper);
    }
```

- 实现类 `io.grpc.services.internal.HealthCheckingRoundRobinLoadBalancerProvider#newLoadBalancer`  

```java
    public LoadBalancer newLoadBalancer(Helper helper) {
        return HealthCheckingLoadBalancerUtil.newHealthCheckingLoadBalancer(rrProvider, helper);
    }
```

- `io.grpc.services.HealthCheckingLoadBalancerUtil#newHealthCheckingLoadBalancer`

```java
    public static LoadBalancer newHealthCheckingLoadBalancer(Factory factory, Helper helper) {
        // 创建工厂
        HealthCheckingLoadBalancerFactory hcFactory = new HealthCheckingLoadBalancerFactory(factory,
                new ExponentialBackoffPolicy.Provider(),
                GrpcUtil.STOPWATCH_SUPPLIER);

        // 使用工厂创建 LoadBalancer
        return hcFactory.newLoadBalancer(helper);
    }
```

- `io.grpc.services.HealthCheckingLoadBalancerFactory#newLoadBalancer`

```java
    public LoadBalancer newLoadBalancer(Helper helper) {
        // 代理 Helper
        HelperImpl wrappedHelper = new HelperImpl(helper);
        // 创建 LoadBalancer
        LoadBalancer delegateBalancer = delegateFactory.newLoadBalancer(wrappedHelper);
        return new HealthCheckingLoadBalancer(wrappedHelper, delegateBalancer);
    }
```

6. 将 `LoadBalancer` 实例作为参数传递给 `NameResolverListener`

- `io.grpc.internal.ManagedChannelImpl#exitIdleMode`

```java
// 服务发现监听器
NameResolverListener listener = new NameResolverListener(lbHelper, nameResolver);
nameResolver.start(listener);
```

## 负载均衡

根据 `NameResolver` 解析的地址，创建相应的 `Subchannel`，在 RPC 请求时根据策略和状态选择其中的一个发起请求

### 处理解析的地址

- `io.grpc.internal.ManagedChannelImpl.NameResolverListener#onResult`

根据解析的结果，获取配置，如果有配置健康检查，则添加健康检查的属性，用于 LB 在连接前进行检查

然后构建参数，调用 `io.grpc.internal.AutoConfiguredLoadBalancerFactory.AutoConfiguredLoadBalancer#tryHandleResolvedAddresses` 方法处理地址

```java
public void run() {

  List<EquivalentAddressGroup> servers = resolutionResult.getAddresses();

  nameResolverBackoffPolicy = null;
  // ...
  ManagedChannelServiceConfig effectiveServiceConfig;
    effectiveServiceConfig = defaultServiceConfig == null ? EMPTY_SERVICE_CONFIG : defaultServiceConfig;
        
  // 获取属性
  Attributes effectiveAttrs = resolutionResult.getAttributes();
  // 如果服务发现没有关闭
  if (NameResolverListener.this.helper == ManagedChannelImpl.this.lbHelper) {
    // 获取健康检查
    Map<String, ?> healthCheckingConfig = effectiveServiceConfig.getHealthCheckingConfig();
    // 构建健康检查配置
    if (healthCheckingConfig != null) {
      effectiveAttrs = effectiveAttrs.toBuilder()
                                     .set(LoadBalancer.ATTR_HEALTH_CHECKING_CONFIG, healthCheckingConfig)
                                     .build();
    }

    // 更新负载均衡算法，处理未处理的请求
    Status handleResult = helper.lb.tryHandleResolvedAddresses(
            ResolvedAddresses.newBuilder()
                             .setAddresses(servers)
                             .setAttributes(effectiveAttrs)
                             .setLoadBalancingPolicyConfig(effectiveServiceConfig.getLoadBalancingConfig())
                             .build());

    if (!handleResult.isOk()) {
      handleErrorInSyncContext(handleResult.augmentDescription(resolver + " was used"));
    }
  }
}
```

### 由 LB 处理地址
- `io.grpc.internal.AutoConfiguredLoadBalancerFactory.AutoConfiguredLoadBalancer#tryHandleResolvedAddresses`

先从解析的结果中获取 `LoadBalancerProvider`，如果不存在，则使用默认的的；
然后获取被代理的 `LoadBalancer`，调用 `handleResolvedAddresses` 方法，由具体的 LB 进行处理

```java
    Status tryHandleResolvedAddresses(ResolvedAddresses resolvedAddresses) {
      List<EquivalentAddressGroup> servers = resolvedAddresses.getAddresses();
      Attributes attributes = resolvedAddresses.getAttributes();

      // 负载均衡选择
      PolicySelection policySelection = (PolicySelection) resolvedAddresses.getLoadBalancingPolicyConfig();

      if (policySelection == null) {
        LoadBalancerProvider defaultProvider;
        // 更新负载均衡提供器
        defaultProvider = getProviderOrThrow(defaultPolicy, "using default policy");
        policySelection = new PolicySelection(defaultProvider, null, null);
      }

      Object lbConfig = policySelection.config;

      // 设置负载均衡算法参数
      if (lbConfig != null) {
        attributes = attributes.toBuilder()
                               .set(ATTR_LOAD_BALANCING_CONFIG, policySelection.rawConfig)
                               .build();
      }

      // 负载均衡器
      LoadBalancer delegate = getDelegate();
      // 如果地址是空的，或者处理失败，则返回错误
      if (resolvedAddresses.getAddresses().isEmpty() && !delegate.canHandleEmptyAddressListFromNameResolution()) {
        return Status.UNAVAILABLE.withDescription("NameResolver returned no usable address. addrs=" + servers + ", attrs=" + attributes);
      } else {
        // 返回处理成功
        delegate.handleResolvedAddresses(
                ResolvedAddresses.newBuilder()
                                 .setAddresses(resolvedAddresses.getAddresses())
                                 .setAttributes(attributes)
                                 .setLoadBalancingPolicyConfig(lbConfig)
                                 .build());
        return Status.OK;
      }
    }
```

- `io.grpc.services.HealthCheckingLoadBalancerFactory.HealthCheckingLoadBalancer#handleResolvedAddresses`

根据配置，获取健康检查的服务名称，然后遍历进行检查
然后调用 `io.grpc.util.RoundRobinLoadBalancer#handleResolvedAddresses` 进行处理

```java
public void handleResolvedAddresses(ResolvedAddresses resolvedAddresses) {
    // 获取健康检查配置
    Map<String, ?> healthCheckingConfig = resolvedAddresses.getAttributes()
                                                           .get(LoadBalancer.ATTR_HEALTH_CHECKING_CONFIG);
    // 获取服务的健康检查配置
    String serviceName = ServiceConfigUtil.getHealthCheckedServiceName(healthCheckingConfig);

    // 配置服务健康检查
    helper.setHealthCheckedService(serviceName);
    // 调用被代理的类处理地址
    super.handleResolvedAddresses(resolvedAddresses);
}
```

- `io.grpc.util.RoundRobinLoadBalancer#handleResolvedAddresses`

在处理地址时，根据现有的地址和新的地址，筛选出需要移除的地址；
然后遍历有效的地址，判断是否已经存在，如果存在，则更新地址集合；如果不存，则调用 `io.grpc.services.HealthCheckingLoadBalancerFactory.HelperImpl#createSubchannel` 创建 Subchannel，启动 `SubchannelStateListener`，监听 Subchannel 状态变化；并调用 `io.grpc.internal.ManagedChannelImpl.SubchannelImpl#requestConnection`要求建立连接

将需要移除的 Subchannel 从集合中移除，更新 LB 状态，并关闭要移除的 Subchannel

```java
public void handleResolvedAddresses(ResolvedAddresses resolvedAddresses) {

    // 获取地址列表
    List<EquivalentAddressGroup> servers = resolvedAddresses.getAddresses();

    // 当前的地址
    Set<EquivalentAddressGroup> currentAddrs = subchannels.keySet();

    // 将地址 List 转为 Map
    Map<EquivalentAddressGroup, EquivalentAddressGroup> latestAddrs = stripAttrs(servers);

    // 根据当前的地址，获取需要移除的地址，返回的地址是现有地址中有，新的地址中没有的
    Set<EquivalentAddressGroup> removedAddrs = setsDifference(currentAddrs, latestAddrs.keySet());


    for (Map.Entry<EquivalentAddressGroup, EquivalentAddressGroup> latestEntry : latestAddrs.entrySet()) {

        // 不含 Attributes 的 EquivalentAddressGroup
        EquivalentAddressGroup strippedAddressGroup = latestEntry.getKey();
        // 包含 Attributes 的 EquivalentAddressGroup
        EquivalentAddressGroup originalAddressGroup = latestEntry.getValue();

        // 根据地址获取对应的已经存在的 Subchannel
        Subchannel existingSubchannel = subchannels.get(strippedAddressGroup);

        // 如果存在已有的 Subchannel，则更新地址并跳出
        if (existingSubchannel != null) {
            // EAG's Attributes may have changed.
            // 更新地址
            existingSubchannel.updateAddresses(Collections.singletonList(originalAddressGroup));
            continue;
        }
        // 根据地址创建新的 Subchannel
        // Create new subchannels for new addresses.

        // 设置新的连接状态是 IDLE
        Attributes.Builder subchannelAttrs = Attributes.newBuilder()
                                                       .set(STATE_INFO, new Ref<>(ConnectivityStateInfo.forNonError(IDLE)));

        // 创建新 Subchannel
        final Subchannel subchannel = checkNotNull(helper.createSubchannel(CreateSubchannelArgs.newBuilder()
                                                                                               .setAddresses(originalAddressGroup)
                                                                                               .setAttributes(subchannelAttrs.build())
                                                                                               .build()),
                "subchannel");

        // 启动 Subchannel 状态监听器
        subchannel.start(new SubchannelStateListener() {
            @Override
            public void onSubchannelState(ConnectivityStateInfo state) {
                // 处理状态变化
                processSubchannelState(subchannel, state);
            }
        });

        // 将新创建的 Subchannel 放在 Subchannel 的 Map 中
        subchannels.put(strippedAddressGroup, subchannel);
        // 要求建立连接
        subchannel.requestConnection();
    }

    // 移除不包含的地址
    ArrayList<Subchannel> removedSubchannels = new ArrayList<>();
    for (EquivalentAddressGroup addressGroup : removedAddrs) {
        removedSubchannels.add(subchannels.remove(addressGroup));
    }

    // 在关闭 Subchannel 之前更新 picker，减少关闭期间的风险
    updateBalancingState();

    // 关闭被移除的 Subchannel
    for (Subchannel removedSubchannel : removedSubchannels) {
        shutdownSubchannel(removedSubchannel);
    }
}
```

### 在请求时做负载均衡

- `io.grpc.internal.ClientCallImpl#startInternal`

在执行 RPC 请求时，调用 `io.grpc.internal.ClientCallImpl#start`，在获取 ClientTransport 时，创建 `PickSubchannelArgsImpl`，通过选择 `Subchannel` 获取 `Transport`

```java
ClientTransport transport = clientTransportProvider.get(new PickSubchannelArgsImpl(method, headers, callOptions));
```

- `io.grpc.internal.ManagedChannelImpl.ChannelTransportProvider#get`

这个方法里，根据状态获取 `Transport`，如果当前的状态是关闭，则直接返回延迟执行的 `Transport`；
如果 Picker 是空的，则说明还没有执行过，则调用 `exitIdleMode` 退出空闲模式，并返回延迟执行的`Transport`;
如果 Picker 已经初始化了，则调用 `io.grpc.util.RoundRobinLoadBalancer.ReadyPicker#pickSubchannel`选择 Subchannel


```java
    public ClientTransport get(PickSubchannelArgs args) {
      SubchannelPicker pickerCopy = subchannelPicker;
      // 如果是关闭状态，则停止调用
      if (shutdown.get()) {
        return delayedTransport;
      }
      
      // 如果是 SubchannelPicker 是空的，则退出 idle 模模式，返回 delayedTransport
      if (pickerCopy == null) {
        final class ExitIdleModeForTransport implements Runnable {
          @Override
          public void run() {
            // 退出 idle 模式，将会创建 LoadBalancer,NameResovler
            exitIdleMode();
          }
        }

        syncContext.execute(new ExitIdleModeForTransport());
        return delayedTransport;
      }
      // 选择某个 SubChannel 发起调用，即选择某个服务端
      PickResult pickResult = pickerCopy.pickSubchannel(args);
      ClientTransport transport = GrpcUtil.getTransportFromPickResult(pickResult, args.getCallOptions().isWaitForReady());
      // 如果有 Transport，则返回
      if (transport != null) {
        return transport;
      }
      return delayedTransport;
    }
```

- `io.grpc.util.RoundRobinLoadBalancer.ReadyPicker#pickSubchannel`

获取下一个 Subchannel 并返回

```java
public PickResult pickSubchannel(PickSubchannelArgs args) {
    return PickResult.withSubchannel(nextSubchannel());
}
```

- `io.grpc.util.RoundRobinLoadBalancer.ReadyPicker#nextSubchannel`

通过轮询的算法获取下一个 Subchannel

```java
private Subchannel nextSubchannel() {
    int size = list.size();
    int i = indexUpdater.incrementAndGet(this);
    if (i >= size) {
        int oldi = i;
        i %= size;
        indexUpdater.compareAndSet(this, oldi, i);
    }
    return list.get(i);
}
```