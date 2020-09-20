---
title: Spring Cloud Kubernetes 服务注册和发现
date: 2020-09-20 22:26:03
tags:
    - Java
    - SpringCloud
categories: 
    - Java
    - SpringCloud
---

# Spring Cloud Kubernetes 服务注册和发现

Spring Cloud Kubernetes 使用，可以通过引入 `org.springframework.cloud:spring-cloud-starter-kubernetes`，这个 starter 依赖于 `org.springframework.cloud:spring-cloud-kubernetes-core` 和 `org.springframework.cloud:spring-cloud-kubernetes-discovery`

## 初始化 Kubernetes Client

### 初始化环境配置 

环境初始化是通过 `org.springframework.cloud.kubernetes.profile.KubernetesProfileEnvironmentPostProcessor`类实现的，当环境初始化完成时，会检查 Kubernetes 是否开启，如果开启则会判断 Profile 是否注入到容器中，没有时将会注入 Profile 到容器中


```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment,
	                                   SpringApplication application) {

		// 判断是否启用 kubernetes，默认为 true
		final boolean kubernetesEnabled = environment.getProperty("spring.cloud.kubernetes.enabled", Boolean.class, true);
		if (!kubernetesEnabled) {
			return;
		}

		// 如果在 Kubernetes 中
		if (isInsideKubernetes()) {
			// 判断是否存在 Kubernetes 环境的配置，如果不存在，则添加到环境变量中
			if (hasKubernetesProfile(environment)) {
				// ...
			} else {
				environment.addActiveProfile(KUBERNETES_PROFILE);
			}
		} else {
			// ...
		}
	}
```


### 初始化 Kubernetes 依赖

相关 Kubernetes 核心依赖的初始化是通过 `org.springframework.cloud.kubernetes.KubernetesAutoConfiguration`实现的

-  初始化 Kubernetes Config

```java
	@Bean
	@ConditionalOnMissingBean(Config.class)
	public Config kubernetesClientConfig(KubernetesClientProperties kubernetesClientProperties) {
		// 先尝试分别加载 ~/.kube 下面的配置，ServiceAccount 和 Namespace 文件
		// 当配置文件中的配置缺失时使用基本的配置
		Config base = Config.autoConfigure(null);
		Config properties = new ConfigBuilder(base)
		//	...
		return properties;		
}
```

加载配置时，会先从本地的 `~/.kube`中寻找配置，根据本地的配置，将当前应用中没有的配置补全，并返回相应的 Bean

- 初始化 Kubernetes Client

```java
	@Bean
	@ConditionalOnMissingBean
	public KubernetesClient kubernetesClient(Config config) {
		return new DefaultKubernetesClient(config);
	}
```

 通过生成的配置初始化 KubernetesClient

## 服务注册

服务注册是通过 `org.springframework.cloud.kubernetes.registry.KubernetesAutoServiceRegistration` 实现的，但是这个类在 2.x 中已经被标记为废弃，因为部署在 Kubernetes 中的服务已经存在于 etcd 中，所以注册并不会真正执行

### 初始化 Bean 

相关 Bean 的初始化是在 `org.springframework.cloud.kubernetes.discovery.KubernetesDiscoveryClientAutoConfiguration` 中完成的

```java
	@Bean
	public KubernetesServiceRegistry getServiceRegistry() {
		return new KubernetesServiceRegistry();
	}
	
	@Bean
	public KubernetesRegistration getRegistration(KubernetesClient client,
	                                              KubernetesDiscoveryProperties properties) {
		return new KubernetesRegistration(client, properties);
	}

	@Bean
	public KubernetesDiscoveryProperties getKubernetesDiscoveryProperties() {
		return new KubernetesDiscoveryProperties();
	}
```

### 注册流程

-  实例化 KubernetesAutoServiceRegistration 类
-  因为实现了 `SmartLifecycle` 接口，所以在应用启动完成，收到 `ServletWebServerInitializedEvent`事件时开始注册

```java
	@EventListener(ServletWebServerInitializedEvent.class)
	public void onApplicationEvent(ServletWebServerInitializedEvent event) {
		int localPort = event.getWebServer().getPort();
		if (this.port.get() == 0) {
			this.port.compareAndSet(0, localPort);
			start();
		}
	}

	@Override
	public void start() {
		this.serviceRegistry.register(this.registration);

		this.context.publishEvent(
				new InstanceRegisteredEvent<>(this, this.registration.getProperties()));
		this.running.set(true);
	}
```

注册完成后发出实例注册的事件

- 在 `KubernetesServiceRegistry` 实现注册逻辑

但实际上并未执行任何注册动作

```java
	@Override
	public void register(KubernetesRegistration registration) {
		log.info("Registering : " + registration);
	}
```

### 取消注册流程

- 监听容器关闭事件 

收到事件后，调用 stop 方法，执行关闭逻辑；在 stop 方法中调用 deregister 方法，取消注册

```java
	@EventListener(ContextClosedEvent.class)
	public void onApplicationEvent(ContextClosedEvent event) {
		if (event.getApplicationContext() == this.context) {
			stop();
		}
	}

	@Override
	public void stop() {
		this.serviceRegistry.deregister(this.registration);
		this.running.set(false);
	}
```

- `KubernetesServiceRegistry` 实现取消注册逻辑

```java
	@Override
	public void deregister(KubernetesRegistration registration) {
		log.info("DeRegistering : " + registration);
	}
```

## 服务发现


### 初始化 Bean 

相关 Bean 的初始化在 `org.springframework.cloud.kubernetes.discovery.KubernetesDiscoveryClient` 中完成

```java
		@Bean
		@ConditionalOnMissingBean
		public KubernetesDiscoveryClient kubernetesDiscoveryClient(
			KubernetesClient client,
			KubernetesDiscoveryProperties properties,
			KubernetesClientServicesFunction kubernetesClientServicesFunction,
			DefaultIsServicePortSecureResolver isServicePortSecureResolver) {
			return new KubernetesDiscoveryClient(client, properties,
				kubernetesClientServicesFunction, isServicePortSecureResolver);
		}
```

###  获取服务

- getService 

调用 `org.springframework.cloud.kubernetes.discovery.KubernetesDiscoveryClient#getServices()` 方法获取指定条件下的服务名称

```java
	@Override
	public List<String> getServices() {
		String spelExpression = this.properties.getFilter();
		Predicate<Service> filteredServices;
		// 根据条件过滤
		if (spelExpression == null || spelExpression.isEmpty()) {
			filteredServices = (Service instance) -> true;
		} else {
			// 解析表达式 并生成过滤条件
			Expression filterExpr = this.parser.parseExpression(spelExpression);
			filteredServices = (Service instance) -> {
				Boolean include = filterExpr.getValue(this.evalCtxt, instance, Boolean.class);
				if (include == null) {
					return false;
				}
				return include;
			};
		}
		return getServices(filteredServices);
	}

	public List<String> getServices(Predicate<Service> filter) {
		return this.kubernetesClientServicesFunction.apply(this.client)
		                                            .list()
		                                            .getItems()
		                                            .stream()
		                                            .filter(filter)
		                                            .map(s -> s.getMetadata().getName())
		                                            .collect(Collectors.toList());
	}
```

### 获取实例 

- getInstance 

调用 `org.springframework.cloud.kubernetes.discovery.KubernetesDiscoveryClient#getInstances`，根据服务的名称获取相应的实例列表

```java
	@Override
	public List<ServiceInstance> getInstances(String serviceId) {
		Assert.notNull(serviceId, "[Assertion failed] - the object argument must not be null");

		// 判断是否查询所有命名空间下的服务，如果是则根据 metadata.name 查询，
		// 否则根据服务名称查询
		List<Endpoints> endpointsList = this.properties.isAllNamespaces()
			? this.client.endpoints()
			             .inAnyNamespace()
			             .withField("metadata.name", serviceId)
			             .list()
			             .getItems()
			: Collections.singletonList(this.client.endpoints().withName(serviceId).get());

		List<EndpointSubsetNS> subsetsNS = endpointsList.stream()
		                                                .map(this::getSubsetsFromEndpoints)
		                                                .collect(Collectors.toList());

		// 获取所有的实例
		List<ServiceInstance> instances = new ArrayList<>();
		if (!subsetsNS.isEmpty()) {
			for (EndpointSubsetNS es : subsetsNS) {
				instances.addAll(this.getNamespaceServiceInstances(es, serviceId));
			}
		}

		return instances;
	}

	private List<ServiceInstance> getNamespaceServiceInstances(EndpointSubsetNS es,
	                                                           String serviceId) {
		String namespace = es.getNamespace();
		List<EndpointSubset> subsets = es.getEndpointSubset();
		List<ServiceInstance> instances = new ArrayList<>();
		if (!subsets.isEmpty()) {
			// 查询指定命名空间下的服务
			final Service service = this.client.services()
			                                   .inNamespace(namespace)
			                                   .withName(serviceId)
			                                   .get();

			// 获取 metadata
			final Map<String, String> serviceMetadata = this.getServiceMetadata(service);
			KubernetesDiscoveryProperties.Metadata metadataProps = this.properties.getMetadata();

			for (EndpointSubset s : subsets) {
				// Extend the service metadata map with per-endpoint port information (if
				// requested)
				Map<String, String> endpointMetadata = new HashMap<>(serviceMetadata);
				if (metadataProps.isAddPorts()) {
					// 获取端口信息
					Map<String, String> ports = s.getPorts()
					                             .stream()
					                             .filter(port -> !StringUtils.isEmpty(port.getName()))
					                             .collect(toMap(EndpointPort::getName, port -> Integer.toString(port.getPort())));
					// 端口数据转为 map
					Map<String, String> portMetadata = getMapWithPrefixedKeys(ports, metadataProps.getPortsPrefix());

					if (log.isDebugEnabled()) {
						log.debug("Adding port metadata: " + portMetadata);
					}

					// 添加到 metadata 中
					endpointMetadata.putAll(portMetadata);
				}

				List<EndpointAddress> addresses = s.getAddresses();
				for (EndpointAddress endpointAddress : addresses) {
					String instanceId = null;
					if (endpointAddress.getTargetRef() != null) {
						instanceId = endpointAddress.getTargetRef().getUid();
					}

					EndpointPort endpointPort = findEndpointPort(s);
					instances.add(new KubernetesServiceInstance(instanceId, serviceId,
						endpointAddress, endpointPort, endpointMetadata,
						this.isServicePortSecureResolver
							.resolve(new DefaultIsServicePortSecureResolver.Input(
								endpointPort.getPort(),
								service.getMetadata().getName(),
								service.getMetadata().getLabels(),
								service.getMetadata().getAnnotations()))));
				}
			}
		}

		return instances;
	}
```

### 服务列表更新

Kubernetes 的服务列表更新是通过定时任务实现的，核心类是 `KubernetesDiscoveryClient` 

Kubernetes 不支持通过服务实例更新，因为调用时是通过 Service 的名称实现的，Kubernetes会做负载均衡，所以不需要在实例维度监听

#### 实例初始化

```java
	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnProperty(name = "spring.cloud.kubernetes.discovery.catalog-services-watch.enabled", matchIfMissing = true)
	public KubernetesCatalogWatch kubernetesCatalogWatch(KubernetesClient client) {
		return new KubernetesCatalogWatch(client);
	}
```

#### 监听实现 

`KubernetesCatalogWatch` 类实现了 `ApplicationEventPublisherAware`接口，用于发现服务列表更新后发送相应的事件

默认执行拉取任务的时间是30s，需要特别注意的是，该任务的开启依赖于`@EnableScheduling`注解开启定时任务，默认不会生效

```java
	@Scheduled(fixedDelayString = "${spring.cloud.kubernetes.discovery.catalogServicesWatchDelay:30000}")
	public void catalogServicesWatch() {
		try {
			List<String> previousState = this.catalogEndpointsState.get();

			// not all pods participate in the service discovery. only those that have
			// endpoints.
			// 仅有 endpoint 的服务参与服务发现
			List<Endpoints> endpoints = this.kubernetesClient.endpoints()
			                                                 .list()
			                                                 .getItems();

			// 将 endpoint 转为pod名称
			List<String> endpointsPodNames = endpoints.stream()
			                                          .map(Endpoints::getSubsets)
			                                          .filter(Objects::nonNull)
			                                          .flatMap(Collection::stream)
			                                          .map(EndpointSubset::getAddresses)
			                                          .filter(Objects::nonNull)
			                                          .flatMap(Collection::stream)
			                                          .map(EndpointAddress::getTargetRef)
			                                          .filter(Objects::nonNull)
			                                          .map(ObjectReference::getName) 
			                                          .sorted(String::compareTo)
			                                          .collect(Collectors.toList());

			this.catalogEndpointsState.set(endpointsPodNames);

			// 如果 pod 列表发生变化，则发送 HeartbeatEvent 事件
			if (!endpointsPodNames.equals(previousState)) {
				logger.trace("Received endpoints update from kubernetesClient: {}", endpointsPodNames);
				this.publisher.publishEvent(new HeartbeatEvent(this, endpointsPodNames));
			}
		} catch (Exception e) {
			logger.error("Error watching Kubernetes Services", e);
		}
	}
```