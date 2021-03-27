---
title: gRPC Server 端启动流程
date: 2020-12-05 22:34:46
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC Server 端启动流程 

gRPC Server 启动流程，底层实现以 Netty 为例；

## 核心类

- `io.grpc.Server` 

Server 的定义接口，实现类是 `io.grpc.internal.ServerImpl`，实现了服务、方法与方法处理器的绑定，端口监听，不同类型的 Server 实现的调用，Server 生命周期管理等


- `io.grpc.BindableService`

由编译器生成的服务抽象类的基础接口，并将实现类绑定到服务器，提供将服务的实现的实例绑定到 Server 的方式

- `io.grpc.ServerInterceptor`

Server 端的拦截器，在方法调用之前会被调用

- `io.grpc.HandlerRegistry`

方法处理器注册器，所有的方法注册器会注册在这里，通过服务名和方法名查找

- `io.grpc.ServerServiceDefinition`

服务定义，包含服务描述信息，方法信息集合

- `io.grpc.ServerMethodDefinition`

方法定义，包含方法描述信息，方法处理器


## 启动流程

### 启动大致流程

1. 创建 `ServerBuilder`
2. 指定端口
3. 为 `ServerBuilder` 添加方法
	1. 构建服务定义
		1. 通过生成的代码构建方法定义，将方法与处理器绑定
		2. 将方法处理器添加到方法定义中
		3. 将服务定义添加到服务注册器中
4. 添加拦截器等其他的配置
5. 构建 `Server` 实例
	1. 构建 `ServerTransport` 实例
		2. 遍历所有监听的地址，创建相应的  `NettyServer`
6. 启动 `Server`
	1. 遍历所有的 `NettyServer`，调用 `start` 方法启动
		1. 创建相应的 `ServerBootstrap`，绑定监听的地址，可以接受连接
		2. 创建 `NettyServerTransport`，调用 `start` 方法启动 `Transport`
		3. 为 `NettyServerTransport` 创建 `NettyServerHandler`，用于处理请求
7. 保持 `Server` 启动状态，启动完成
	

### Server 定义

```java
Server server = ServerBuilder.forPort(1234) 
                             .addService(new HelloServiceImpl()) 
                             .intercept(new CustomServerInterceptor())
                             .build();
server.start();
server.awaitTermination();
```

#### 绑定端口

指定了要监听的端口，会根据不同的 `Server` 实现绑定端口

- io.grpc.ServerBuilder#forPort

```java
public static ServerBuilder<?> forPort(int port) {
    return ServerProvider.provider().builderForPort(port);
}
```

- io.grpc.netty.NettyServerProvider#builderForPort

```java
protected NettyServerBuilder builderForPort(int port) {
  return NettyServerBuilder.forPort(port);
}
``` 

- io.grpc.netty.NettyServerBuilder#NettyServerBuilder(int)

最终会使用指定的端口，创建 `InetSocketAddress` 并将其加入到监听的地址集合中

```java
private NettyServerBuilder(int port) {
    // 将本地 IP 和端口的地址添加到监听的地址集合中
    this.listenAddresses.add(new InetSocketAddress(port));
}
```


#### 绑定服务

将指定的服务实现类添加到方法注册器中

- io.grpc.internal.AbstractServerImplBuilder#addService(io.grpc.BindableService)

添加的服务是 `BindableService` 接口的实现类的实例

```java
public final T addService(BindableService bindableService) {
    return addService(checkNotNull(bindableService, "bindableService").bindService());
}
```

- io.github.helloworlde.HelloServiceGrpc.HelloServiceImplBase#bindService

这个方法是生成的代码，里面创建了服务定义、并绑定了方法与处理器

```java
public final io.grpc.ServerServiceDefinition bindService() {
    return io.grpc.ServerServiceDefinition.builder(getServiceDescriptor())
                                          .addMethod(getHowAreYouMethod(), asyncUnaryCall(new MethodHandlers<io.github.helloworlde.HelloMessage, io.github.helloworlde.HelloResponse>(this, METHODID_HOW_ARE_YOU)))
                                          .build();
}
```

其中 `addMethod` 会获取方法，为方法绑定处理器，即当前的实现类，并构建方法定义；其中 `getHowAreYouMethod`用于获取方法定义，最终返回包含方法类型、全名、编解码等方法描述信息的实例

- io.github.helloworlde.HelloServiceGrpc.MethodHandlers#MethodHandlers

通过方法实现类实例和方法的 ID 构建 `MethodHandlers`实例，这个方法也是生成的，在处理请求时会根据方法的 ID 选择不同的实现类调用相应的方法处理请求

```java
MethodHandlers(HelloServiceImplBase serviceImpl, int methodId) {
    this.serviceImpl = serviceImpl;
    this.methodId = methodId;
}

@java.lang.Override
public void invoke(Req request, io.grpc.stub.StreamObserver<Resp> responseObserver) {
    switch (methodId) {
        case METHODID_HOW_ARE_YOU:
            serviceImpl.howAreYou((io.github.helloworlde.HelloMessage) request,
                    (io.grpc.stub.StreamObserver<io.github.helloworlde.HelloResponse>) responseObserver);
            break;
        default:
            throw new AssertionError();
    }
}
```

- io.grpc.stub.ServerCalls#asyncUnaryCall

会根据构建的 `MethodHandlers`  创建相应类型请求的处理器，并将其加入到服务定义中

```java
public static <ReqT, RespT> ServerCallHandler<ReqT, RespT> asyncUnaryCall(UnaryMethod<ReqT, RespT> method) {
    return new UnaryServerCallHandler<>(method);
}
```

创建的 `UnaryServerCallHandler` 会在服务端 `onHalfClose` 时调用构建的 `UnaryMethod`的`invoke`方法，处理请求


- io.grpc.ServerServiceDefinition.Builder#build

遍历服务中的所有的方法，构建服务定义

```
public ServerServiceDefinition build() {
    ServiceDescriptor serviceDescriptor = this.serviceDescriptor;
    // 如果服务定义为 null，则遍历方法，用服务名和方法集合构建新的服务定义
    if (serviceDescriptor == null) {
        List<MethodDescriptor<?, ?>> methodDescriptors = new ArrayList<>(methods.size());
        for (ServerMethodDefinition<?, ?> serverMethod : methods.values()) {
            methodDescriptors.add(serverMethod.getMethodDescriptor());
        }
        serviceDescriptor = new ServiceDescriptor(serviceName, methodDescriptors);
    }

    Map<String, ServerMethodDefinition<?, ?>> tmpMethods = new HashMap<>(methods);
    // 遍历方法定义，校验是否所有的方法都在方法定义中
    for (MethodDescriptor<?, ?> descriptorMethod : serviceDescriptor.getMethods()) {
        ServerMethodDefinition<?, ?> removed = tmpMethods.remove(descriptorMethod.getFullMethodName());
    }

    // 根据服务定义和方法定义集合构建服务定义
    return new ServerServiceDefinition(serviceDescriptor, methods);
}
```

- io.grpc.internal.AbstractServerImplBuilder#addService(io.grpc.ServerServiceDefinition)

将服务定义添加到服务注册器中

```java
public final T addService(ServerServiceDefinition service) {
    // 将服务添加到处理器中
    registryBuilder.addService(checkNotNull(service, "service"));
    return thisT();
}
```

- io.grpc.internal.InternalHandlerRegistry.Builder#addService

最终将服务定义添加到注册器的 Map 中，key 是服务的全名，value 是服务定义；
这样就可以在处理请求时根据请求中的服务和方法名称，在注册器中查找对应的方法的处理器，实现调用

```java
Builder addService(ServerServiceDefinition service) {
    services.put(service.getServiceDescriptor().getName(), service);
    return this;
}
```

#### 绑定拦截器

- io.grpc.internal.AbstractServerImplBuilder#intercept

通过 `intercept` 方法，将拦截器添加到拦截器集合中

```java
public final T intercept(ServerInterceptor interceptor) {
    interceptors.add(checkNotNull(interceptor, "interceptor"));
    return thisT();
}
```

#### 构建 Server 实例

- io.grpc.internal.AbstractServerImplBuilder#build

在构建器中创建 `Server` 的实例

```java
public final Server build() {
    return new ServerImpl(this, buildTransportServers(getTracerFactories()), Context.ROOT);
}
```

- io.grpc.netty.NettyServerBuilder#buildTransportServers

构建 `Server` 的 `Transport`，会为每个监听的地址都构建一个 `NettyServer`

```java
protected List<NettyServer> buildTransportServers(List<? extends ServerStreamTracer.Factory> streamTracerFactories) {
    assertEventLoopsAndChannelType();

    ProtocolNegotiator negotiator = protocolNegotiator;
    if (negotiator == null) {
        negotiator = sslContext != null ? ProtocolNegotiators.serverTls(sslContext, this.getExecutorPool()) : ProtocolNegotiators.serverPlaintext();
    }

    List<NettyServer> transportServers = new ArrayList<>(listenAddresses.size());
    // 为每一个监听的地址构建一个 NettyServer
    for (SocketAddress listenAddress : listenAddresses) {
        NettyServer transportServer = new NettyServer(listenAddress,
                channelFactory,
                channelOptions,
                childChannelOptions,
                bossEventLoopGroupPool,
                workerEventLoopGroupPool,
                forceHeapBuffer,
                negotiator,
                streamTracerFactories,
                getTransportTracerFactory(),
                maxConcurrentCallsPerConnection,
                autoFlowControl,
                flowControlWindow,
                maxMessageSize,
                maxHeaderListSize,
                keepAliveTimeInNanos,
                keepAliveTimeoutInNanos,
                maxConnectionIdleInNanos,
                maxConnectionAgeInNanos,
                maxConnectionAgeGraceInNanos,
                permitKeepAliveWithoutCalls,
                permitKeepAliveTimeInNanos,
                getChannelz());
        transportServers.add(transportServer);
    }
    return Collections.unmodifiableList(transportServers);
}
```

- io.grpc.internal.ServerImpl#ServerImpl

创建 `ServerImpl` 实例

```java
ServerImpl(AbstractServerImplBuilder<?> builder,
           List<? extends InternalServer> transportServers,
           Context rootContext) {
    this.executorPool = Preconditions.checkNotNull(builder.executorPool, "executorPool");
    this.registry = Preconditions.checkNotNull(builder.registryBuilder.build(), "registryBuilder");
    this.fallbackRegistry = Preconditions.checkNotNull(builder.fallbackRegistry, "fallbackRegistry");
    Preconditions.checkNotNull(transportServers, "transportServers");
    Preconditions.checkArgument(!transportServers.isEmpty(), "no servers provided");
    this.transportServers = new ArrayList<>(transportServers);
    this.logId = InternalLogId.allocate("Server", String.valueOf(getListenSocketsIgnoringLifecycle()));
    // Fork from the passed in context so that it does not propagate cancellation, it only
    // inherits values.
    this.rootContext = Preconditions.checkNotNull(rootContext, "rootContext").fork();
    this.decompressorRegistry = builder.decompressorRegistry;
    this.compressorRegistry = builder.compressorRegistry;
    this.transportFilters = Collections.unmodifiableList(new ArrayList<>(builder.transportFilters));
    this.interceptors = builder.interceptors.toArray(new ServerInterceptor[builder.interceptors.size()]);
    this.handshakeTimeoutMillis = builder.handshakeTimeoutMillis;
    this.binlog = builder.binlog;
    this.channelz = builder.channelz;
    this.serverCallTracer = builder.callTracerFactory.create();
    this.ticker = checkNotNull(builder.ticker, "ticker");
    channelz.addServer(this);
}
```

### 启动 Server 

#### 启动 Server

- io.grpc.internal.ServerImpl#start

启动 `Server`，会创建服务监听器，遍历所有的监听的地址，并启动相应的 `Transport`，修改启动状态为 true

```java
public ServerImpl start() throws IOException {
    synchronized (lock) {
        checkState(!started, "Already started");
        checkState(!shutdown, "Shutting down");
        // Start and wait for any ports to actually be bound.
        // 创建服务监听器实例
        ServerListenerImpl listener = new ServerListenerImpl();
        // 遍历 Server 并启动
        for (InternalServer ts : transportServers) {
            ts.start(listener);
            activeTransportServers++;
        }
        executor = Preconditions.checkNotNull(executorPool.getObject(), "executor");
        started = true;
        return this;
    }
}
```

- io.grpc.netty.NettyServer#start

启动 Netty Server，会先创建 `ServerBootstrap`，然后在初始化 `Channel` 时会创建  `NettyServerTransport` 并调用其 start 方法启动，并将需要监听的地址绑定在  `ServerBootstrap` 上

```java
@Override
public void start(ServerListener serverListener) throws IOException {
  listener = checkNotNull(serverListener, "serverListener");

  // 用于启动 Netty ServerChannel
  ServerBootstrap b = new ServerBootstrap();
  b.group(bossGroup, workerGroup);

  b.childHandler(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(Channel ch) {

      ChannelPromise channelDone = ch.newPromise();

      // 构建基于 Netty 的 ServerTransport
      NettyServerTransport transport = new NettyServerTransport(ch,
              channelDone,
              protocolNegotiator,
              streamTracerFactories,
              transportTracerFactory.create(),
              maxStreamsPerConnection,
              autoFlowControl,
              flowControlWindow,
              maxMessageSize,
              maxHeaderListSize,
              keepAliveTimeInNanos,
              keepAliveTimeoutInNanos,
              maxConnectionIdleInNanos,
              maxConnectionAgeInNanos,
              maxConnectionAgeGraceInNanos,
              permitKeepAliveWithoutCalls,
              permitKeepAliveTimeInNanos);
      ServerTransportListener transportListener;
      synchronized (NettyServer.this) {
        // 调用监听器回调，Transport 创建事件
        transportListener = listener.transportCreated(transport);
      }

      // 启动监听器
      transport.start(transportListener);
    }
  });
  // 绑定地址，接受连接
  ChannelFuture future = b.bind(address);

  channel = future.channel();
  channel.eventLoop().execute(new Runnable() {
    @Override
    public void run() {
      listenSocketStats = new ListenSocket(channel);
      channelz.addListenSocket(listenSocketStats);
    }
  });
}
```

- io.grpc.internal.ServerImpl.ServerListenerImpl#transportCreated

`ServerTransport` 创建事件回调

```java
@Override
public ServerTransportListener transportCreated(ServerTransport transport) {
    synchronized (lock) {
        transports.add(transport);
    }
    // Transport 监听器
    ServerTransportListenerImpl stli = new ServerTransportListenerImpl(transport);
    // 初始化监听器，添加握手超时的取消任务
    stli.init();
    return stli;
}
```

- io.grpc.netty.NettyServerTransport#start

启动 `NettyServerTransport`，会创建相应的 `NettyServerHandler` 用于处理请求；同时添加监听器

```java
public void start(ServerTransportListener listener) {
    Preconditions.checkState(this.listener == null, "Handler already registered");
    this.listener = listener;

    // Create the Netty handler for the pipeline.
    // 为 pipeline 创建 Netty Handler
    grpcHandler = createHandler(listener, channelUnused);

    // Notify when the channel closes.
    // 当 Channel 关闭的时候发送通知
    final class TerminationNotifier implements ChannelFutureListener {
        boolean done;

        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            if (!done) {
                done = true;
                notifyTerminated(grpcHandler.connectionError());
            }
        }
    }

    // 创建 Handler
    ChannelHandler negotiationHandler = protocolNegotiator.newHandler(grpcHandler);
    ChannelHandler bufferingHandler = new WriteBufferingAndExceptionHandler(negotiationHandler);

    // 添加监听器
    ChannelFutureListener terminationNotifier = new TerminationNotifier();
    channelUnused.addListener(terminationNotifier);
    channel.closeFuture().addListener(terminationNotifier);

    channel.pipeline().addLast(bufferingHandler);
}
```

#### 保持 Server 运行

- io.grpc.internal.ServerImpl#awaitTermination()

通过轮询关闭的状态，如果没有关闭，则使锁等待，保持 Server 线程的运行

```java
public void awaitTermination() throws InterruptedException {
    synchronized (lock) {
        while (!terminated) {
            lock.wait();
        }
    }
}
```

