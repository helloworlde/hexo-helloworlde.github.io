---
title: Spring Cloud Consul 服务注册和发现
date: 2020-09-20 22:27:16
tags:
    - Java
    - SpringCloud
categories: 
    - Java
    - SpringCloud
---

# Spring Cloud Consul 服务注册和发现

Spring Cloud Kubernetes 使用，可以通过引入 `org.springframework.cloud:spring-cloud-starter-consul-discovery`，这个 starter 依赖于 `org.springframework.cloud:spring-cloud-consul-core` 和 `org.springframework.cloud:spring-cloud-consul-discovery`

## Consul 的核心概念

- server 
集群的核心节点，用于和 agent 通讯，保存服务的信息

- agent
集群节点的守护进程，用于服务注册等行为，但不保存数据

- catalog
集群服务通信的接口

## 初始化 Kubernetes Client

### 初始化 Consul 依赖

相关 Consul 核心依赖的初始化是通过 `org.springframework.cloud.consul.ConsulAutoConfiguration`实现的

-  初始化 ConsulClient

```java
	@Bean
	@ConditionalOnMissingBean
	public ConsulClient consulClient(ConsulProperties consulProperties) {
		final int agentPort = consulProperties.getPort();
		final String agentHost = !StringUtils.isEmpty(consulProperties.getScheme())
			? consulProperties.getScheme() + "://" + consulProperties.getHost()
			: consulProperties.getHost();

		if (consulProperties.getTls() != null) {
			ConsulProperties.TLSConfig tls = consulProperties.getTls();
			TLSConfig tlsConfig = new TLSConfig(tls.getKeyStoreInstanceType(),
				tls.getCertificatePath(), tls.getCertificatePassword(),
				tls.getKeyStorePath(), tls.getKeyStorePassword());
			return new ConsulClient(agentHost, agentPort, tlsConfig);
		}
		return new ConsulClient(agentHost, agentPort);
	}
```



## 服务注册

### 初始化 Bean 

相关 Bean 的初始化是在 `org.springframework.cloud.consul.serviceregistry.ConsulAutoServiceRegistrationAutoConfiguration` 中完成的

```java
    // 自动注册
	@Bean
	@ConditionalOnMissingBean
	public ConsulAutoServiceRegistration consulAutoServiceRegistration(
		ConsulServiceRegistry registry,
		AutoServiceRegistrationProperties autoServiceRegistrationProperties,
		ConsulDiscoveryProperties properties,
		ConsulAutoRegistration consulRegistration) {
		return new ConsulAutoServiceRegistration(registry, autoServiceRegistrationProperties, properties, consulRegistration);
	}

	// 启动事件监听器
	@Bean
	public ConsulAutoServiceRegistrationListener consulAutoServiceRegistrationListener(
		ConsulAutoServiceRegistration registration) {
		return new ConsulAutoServiceRegistrationListener(registration);
	}
	@Bean
	@ConditionalOnMissingBean
	public ConsulAutoRegistration consulRegistration(
		AutoServiceRegistrationProperties autoServiceRegistrationProperties,
		ConsulDiscoveryProperties properties, ApplicationContext applicationContext,
		ObjectProvider<List<ConsulRegistrationCustomizer>> registrationCustomizers,
		ObjectProvider<List<ConsulManagementRegistrationCustomizer>> managementRegistrationCustomizers,
		HeartbeatProperties heartbeatProperties) {
		return ConsulAutoRegistration.registration(autoServiceRegistrationProperties,
			properties, applicationContext, registrationCustomizers.getIfAvailable(),
			managementRegistrationCustomizers.getIfAvailable(), heartbeatProperties);
	}
```

### 注册流程

- 当监听到 `WebServerInitializedEvent` 事件时触发注册

`ConsulAutoServiceRegistrationListener` 类实现了 `SmartApplicationListener`接口

```java
	@Override
	public void onApplicationEvent(ApplicationEvent applicationEvent) {
		// 判断是否是 web server 初始化事件
		if (applicationEvent instanceof WebServerInitializedEvent) {
			WebServerInitializedEvent event = (WebServerInitializedEvent) applicationEvent;

			ApplicationContext context = event.getApplicationContext();
			if (context instanceof ConfigurableWebServerApplicationContext) {
				if ("management".equals(((ConfigurableWebServerApplicationContext) context).getServerNamespace())) {
					return;
				}
			}
			this.autoServiceRegistration.setPortIfNeeded(event.getWebServer().getPort());
			// 真正触发服务注册
			this.autoServiceRegistration.start();
		}
	}
```

调用 `org.springframework.cloud.consul.serviceregistry.ConsulAutoServiceRegistration#register` 注册

```java
	@Override
	protected void register() {
		if (!this.properties.isRegister()) {
			log.debug("Registration disabled.");
			return;
		}

		super.register();
	}
```

然后调用 `org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#start`

```java
    public void start() {
        if (!this.isEnabled()) {
            if (logger.isDebugEnabled()) {
                logger.debug("Discovery Lifecycle disabled. Not starting");
            }

        } else {
            if (!this.running.get()) {
                this.context.publishEvent(new InstancePreRegisteredEvent(this, this.getRegistration()));
                this.register();
                if (this.shouldRegisterManagement()) {
                    this.registerManagement();
                }

                this.context.publishEvent(new InstanceRegisteredEvent(this, this.getConfiguration()));
                this.running.compareAndSet(false, true);
            }

        }
    }

    protected void register() {
        this.serviceRegistry.register(this.getRegistration());
    }
```


- 最终在 `ConsulServiceRegistry` 实现注册逻辑

```java
	@Override
	public void register(ConsulRegistration reg) {
		log.info("Registering service with consul: " + reg.getService());
		try {
			// 将服务注册到 Consul
			this.client.agentServiceRegister(reg.getService(), this.properties.getAclToken());
			NewService service = reg.getService();

			// 添加到心跳检查中
			if (this.heartbeatProperties.isEnabled() &&
				this.ttlScheduler != null &&
				service.getCheck() != null &&
				service.getCheck().getTtl() != null) {
				this.ttlScheduler.add(reg.getInstanceId());
			}
		} catch (ConsulException e) {
			if (this.properties.isFailFast()) {
				log.error("Error registering service with consul: " + reg.getService(), e);
				ReflectionUtils.rethrowRuntimeException(e);
			}
			log.warn("Failfast is false. Error registering service with consul: " + reg.getService(), e);
		}
	}
```

最后发出服务注册事件

### 取消注册流程

- 在 Bean `org.springframework.cloud.consul.serviceregistry.ConsulAutoServiceRegistration` 销毁的时候调用 stop 方法，执行关闭逻辑；在 stop 方法中调用 deregister 方法，取消注册

```java
    public void stop() {
        if (this.getRunning().compareAndSet(true, false) && this.isEnabled()) {
            this.deregister();
            if (this.shouldRegisterManagement()) {
                this.deregisterManagement();
            }

            this.serviceRegistry.close();
        }

    }
```

- `ConsulServiceRegistry` 实现取消注册逻辑

```java
	@Override
	public void deregister(ConsulRegistration reg) {
		if (this.ttlScheduler != null) {
			this.ttlScheduler.remove(reg.getInstanceId());
		}
		if (log.isInfoEnabled()) {
			log.info("Deregistering service with consul: " + reg.getInstanceId());
		}
		// 将实例从 Consul 中移除
		this.client.agentServiceDeregister(reg.getInstanceId(), this.properties.getAclToken());
	}

```

## 服务发现


### 初始化 Bean 

相关 Bean 的初始化在 `org.springframework.cloud.consul.discovery.ConsulDiscoveryClientConfiguration` 中完成

```java
	@Bean
	@ConditionalOnMissingBean
	public ConsulDiscoveryClient consulDiscoveryClient(ConsulClient consulClient,
	                                                   ConsulDiscoveryProperties discoveryProperties) {
		return new ConsulDiscoveryClient(consulClient, discoveryProperties);
	}
```

###  获取服务

- getService 

调用 `org.springframework.cloud.consul.discovery.ConsulDiscoveryClient#getServices` 方法获取指定条件下的服务名称

```java
	@Override
	public List<String> getServices() {
		String aclToken = this.properties.getAclToken();

		CatalogServicesRequest request = CatalogServicesRequest.newBuilder()
		                                                       .setQueryParams(QueryParams.DEFAULT)
		                                                       .setToken(this.properties.getAclToken()).build();
		return new ArrayList<>(this.client.getCatalogServices(request).getValue().keySet());
	}
```

最终是调用了 Consul 的 `/v1/catalog/services`接口

### 获取实例 

- getInstance 

调用 `org.springframework.cloud.consul.discovery.ConsulDiscoveryClient#getInstances(java.lang.String, com.ecwid.consul.v1.QueryParams)`，根据服务的名称获取相应的实例列表

```java
	public List<ServiceInstance> getInstances(final String serviceId,
	                                          final QueryParams queryParams) {
		List<ServiceInstance> instances = new ArrayList<>();

		addInstancesToList(instances, serviceId, queryParams);

		return instances;
	}


	private void addInstancesToList(List<ServiceInstance> instances, String serviceId,
	                                QueryParams queryParams) {

		// 请求参数
		HealthServicesRequest request = HealthServicesRequest.newBuilder()
		                                                     .setTag(this.properties.getDefaultQueryTag())
		                                                     .setPassing(this.properties.isQueryPassing())
		                                                     .setQueryParams(queryParams)
		                                                     .setToken(this.properties.getAclToken()).build();
		Response<List<HealthService>> services = this.client.getHealthServices(serviceId,
			request);

		for (HealthService service : services.getValue()) {
			String host = findHost(service);

			Map<String, String> metadata = getMetadata(service, this.properties.isTagsAsMetadata());
			boolean secure = false;
			if (metadata.containsKey("secure")) {
				secure = Boolean.parseBoolean(metadata.get("secure"));
			}
			instances.add(
				new DefaultServiceInstance(
					service.getService().getId(),
					serviceId,
					host,
					service.getService().getPort(),
					secure,
					metadata)
			);
		}
	}

```

## 服务列表更新

Consul 的实例监听是通过定时任务，默认每秒都会拉取服务列表，如果发现返回的 Index 发生变化，则说明服务发生变化，发出 `HeartbeatEvent` 事件

### 实例初始化

```java
	@Bean
	@ConditionalOnMissingBean
	public ConsulCatalogWatch consulCatalogWatch(
			ConsulDiscoveryProperties discoveryProperties, 
			ConsulClient consulClient,
			@Qualifier(CATALOG_WATCH_TASK_SCHEDULER_NAME) TaskScheduler taskScheduler) {
		return new ConsulCatalogWatch(discoveryProperties, consulClient, taskScheduler);
	}

```

### 监听实现

是在 `org.springframework.cloud.consul.discovery.ConsulCatalogWatch#catalogServicesWatch`

```java
	@Timed("consul.watch-catalog-services")
	public void catalogServicesWatch() {
		try {
			long index = -1;
			if (this.catalogServicesIndex.get() != null) {
				index = this.catalogServicesIndex.get().longValue();
			}

			// 获取服务信息
			CatalogServicesRequest request = CatalogServicesRequest.newBuilder()
			                                                       .setQueryParams(new QueryParams(this.properties.getCatalogServicesWatchTimeout(), index))
			                                                       .setToken(this.properties.getAclToken()).build();
			Response<Map<String, List<String>>> response = this.consul.getCatalogServices(request);
			// 获取位点并发送事件
			Long consulIndex = response.getConsulIndex();
			if (consulIndex != null) {
				this.catalogServicesIndex.set(BigInteger.valueOf(consulIndex));
			}

			if (log.isTraceEnabled()) {
				log.trace("Received services update from consul: " + response.getValue() + ", index: " + consulIndex);
			}
			this.publisher.publishEvent(new HeartbeatEvent(this, consulIndex));
		} catch (Exception e) {
			log.error("Error watching Consul CatalogServices", e);
		}
	}

```