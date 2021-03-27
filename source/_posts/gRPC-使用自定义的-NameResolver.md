---
title: gRPC 使用自定义的 NameResolver
date: 2020-09-20 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 使用自定义的 NameResolver

在使用注册中心时，gRPC 并未提供注册中心的服务发现，需要自己实现 `NameResolverProvider` 和 `NameResolver`

- NameResolver

`NameResolver` 里面重写了 `start` 和 `refresh` 方法，这两个方法都调用一个 `resolve` 方法做服务发现；
`resovle` 方法内部通过服务名从注册中心拉取服务实例列表，然后调用 `Listener` 的 `onResult`方法，将实例列表传递给 `LoadBalancer`，完成服务解析
在服务运行期间，因为实例可能会发生变化，所以可以通过定时执行触发服务解析；如果注册中心支持，也可以通过回调触发

```java
public class CustomNameResolver extends NameResolver {

    private final ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(10);

    private final String authority;

    private Listener2 listener;

    public CustomNameResolver(String authority) {
        this.authority = authority;
    }

    @Override
    public String getServiceAuthority() {
        return this.authority;
    }

    @Override
    public void shutdown() {
        this.executorService.shutdown();
    }

    @Override
    public void start(Listener2 listener) {
        this.listener = listener;
        this.resolve();
    }

    @Override
    public void refresh() {
        this.resolve();
    }

    private void resolve() {
        executorService.scheduleAtFixedRate(() -> {
            //    从注册中心获取地址
            List<InetSocketAddress> addressList = getAddressList(this.authority);
            if (addressList == null || addressList.size() == 0) {
                return;
            }
            List<SocketAddress> socketAddressList = addressList.stream()
                                                               .map(this::toSocketAddress)
                                                               .collect(Collectors.toList());
            EquivalentAddressGroup equivalentAddressGroup = new EquivalentAddressGroup(socketAddressList);

            Attributes.Key<String> key = Attributes.Key.create("CustomKey");
            String value = "CustomValue";

            Attributes attributes = Attributes.newBuilder().set(key, value).build();
            ConfigOrError configOrError = ConfigOrError.fromError(Status.NOT_FOUND);

            ResolutionResult resolutionResult = ResolutionResult.newBuilder()
                                                                .setAddresses(Arrays.asList(equivalentAddressGroup))
                                                                .setAttributes(attributes)
                                                                .setServiceConfig(configOrError)
                                                                .build();

            this.listener.onResult(resolutionResult);
        }, 0, 5, TimeUnit.SECONDS);

    }

    private SocketAddress toSocketAddress(InetSocketAddress address) {
        return new InetSocketAddress(address.getHostName(), address.getPort());
    }

    private List<InetSocketAddress> getAddressList(String authority) {
        InetSocketAddress inetSocketAddress = new InetSocketAddress("localhost", 1234);
        InetSocketAddress inetSocketAddress2 = new InetSocketAddress("127.0.0.1", 1234);
        return Arrays.asList(inetSocketAddress, inetSocketAddress2);
    }
}
```

- NameResolverProvider 

`NameResolverProvider` 主要用于注册 `NameResolver`，可以设置默认的协议，是否可用，优先级等
优先级有效值是 0-10，gRPC 默认的 `DnsNameResolver` 优先级是5，所以自定义的优先级要大于5

```java
public class CustomNameResolverProvider extends NameResolverProvider {
    @Override
    public NameResolver newNameResolver(URI targetUri, NameResolver.Args args) {
        return new CustomNameResolver(targetUri.toString());
    }

    @Override
    protected boolean isAvailable() {
        return true;
    }

    @Override
    protected int priority() {
        return 10;
    }

    @Override
    public String getDefaultScheme() {
        return "http";
    }
}
```

- 注册 NameResolver 

在新版本(1.21+)之后，`NameResolver` 推荐使用 `NameResolverRegistry` 进行注册；
注册要在创建 Channel 之前执行

```java
NameResolverRegistry.getDefaultRegistry().register(new CustomNameResolverProvider());

this.channel = ManagedChannelBuilder
                .forTarget("server")
                .usePlaintext()
                .build();

```

在 Channel 调用 `build` 方式时，会在 `io.grpc.internal.ManagedChannelImpl#ManagedChannelImpl`的构造方法中获取 `NameResolver.Factory`，这个属性的值是由调用 `io.grpc.internal.AbstractManagedChannelImplBuilder#getNameResolverFactory` 方法获取的，这个方法里面的属性值来自于 `io.grpc.NameResolverRegistry#asFactory`；`NameResolverRegistry` 自己通过内部类 `NameResolverFactory`创建了`NameResovler.Factory` 的实例，调用 Factory 的 `newNameResolver`时，从 `provider` 属性中获取根据优先级排序后的 `NameResolver`，创建实例并返回第一个创建的有效实例