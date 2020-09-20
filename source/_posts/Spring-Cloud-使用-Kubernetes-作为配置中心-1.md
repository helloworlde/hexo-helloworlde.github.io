---
title: Spring Cloud 使用 Kubernetes 作为配置中心
date: 2020-09-20 22:28:15
tags:
    - Java
    - SpringCloud
categories: 
    - Java
    - SpringCloud
---

# Spring Cloud 使用 Kubernetes 作为配置中心

Spring Cloud 支持使用 Kubernetes 作为配置中心，通过 ConfigMap 或 Secret，将配置添加到应用中

## 加载配置 

加载配置是通过 PropertySourceLocator 来实现的，ConfigMap 使用 `ConfigMapPropertySourceLocator` 加载，Secret 使用 `SecretsPropertySourceLocator`加载

### Bean 初始化

```java
		@Bean
		@ConditionalOnProperty(name = "spring.cloud.kubernetes.config.enabled", matchIfMissing = true)
		public ConfigMapPropertySourceLocator configMapPropertySourceLocator(ConfigMapConfigProperties properties) {
			return new ConfigMapPropertySourceLocator(this.client, properties);
		}

		@Bean
		@ConditionalOnProperty(name = "spring.cloud.kubernetes.secrets.enabled", matchIfMissing = true)
		public SecretsPropertySourceLocator secretsPropertySourceLocator(SecretsConfigProperties properties) {
			return new SecretsPropertySourceLocator(this.client, properties);
		}
```

### 获取配置

获取配置是通过 PropertySourceLocator#locate 方法实现的，最终将获取到属性添加到环境中

#### ConfigMap 

- ConfigMapPropertySourceLocator#locate

```java
	@Override
	public PropertySource locate(Environment environment) {
		if (environment instanceof ConfigurableEnvironment) {
			ConfigurableEnvironment env = (ConfigurableEnvironment) environment;

			List<ConfigMapConfigProperties.NormalizedSource> sources = this.properties.determineSources();
			CompositePropertySource composite = new CompositePropertySource("composite-configmap");
			if (this.properties.isEnableApi()) {
				sources.forEach(s -> composite.addFirstPropertySource(
					getMapPropertySourceForSingleConfigMap(env, s)));
			}

			// 将配置添加的容器环境中
			addPropertySourcesFromPaths(environment, composite);

			return composite;
		}
		return null;
	}
```

真正向 Kubernetes 发起请求的是通过调用 `getMapPropertySourceForSingleConfigMap` 方法，创建`ConfigMapPropertySource`实例的时候，会根据 `getData` 方法，从 ConfigMap 获取属性解析并添加到环境中

- ConfigMapPropertySourceLocator#getMapPropertySourceForSingleConfigMap

```java
	private MapPropertySource getMapPropertySourceForSingleConfigMap(
		ConfigurableEnvironment environment, NormalizedSource normalizedSource) {

		String configurationTarget = this.properties.getConfigurationTarget();
		// 创建新的属性
		return new ConfigMapPropertySource(this.client,
			getApplicationName(environment, normalizedSource.getName(), configurationTarget),
			getApplicationNamespace(this.client, normalizedSource.getNamespace(), configurationTarget),
			environment);
	}
```

- ConfigMapPropertySource

```java
	public ConfigMapPropertySource(KubernetesClient client, 
	                               String name, 
	                               String namespace,
	                               Environment environment) {
		super(getName(client, name, namespace),
			asObjectMap(getData(client, name, namespace, environment)));
	}
```

- ConfigMapPropertySource#getData

```java
	private static Map<String, Object> getData(KubernetesClient client, String name,
	                                           String namespace, Environment environment) {
		try {
			Map<String, Object> result = new LinkedHashMap<>();
			// 获取 ConfigMap
			ConfigMap map = StringUtils.isEmpty(namespace)
				? client.configMaps().withName(name).get()
				: client.configMaps().inNamespace(namespace).withName(name).get();

			// 添加到map 中
			if (map != null) {
				result.putAll(processAllEntries(map.getData(), environment));
			}

			if (environment != null) {
				// 根据 Profile 加载 ConfigMap
				for (String activeProfile : environment.getActiveProfiles()) {

					String mapNameWithProfile = name + "-" + activeProfile;

					ConfigMap mapWithProfile = StringUtils.isEmpty(namespace)
						? client.configMaps().withName(mapNameWithProfile).get()
						: client.configMaps().inNamespace(namespace)
						        .withName(mapNameWithProfile).get();

					if (mapWithProfile != null) {
						result.putAll(
							processAllEntries(mapWithProfile.getData(), environment));
					}

				}
			}

			return result;

		} catch (Exception e) {
			LOG.warn("Can't read configMap with name: [" + name + "] in namespace:["
				+ namespace + "]. Ignoring.", e);
		}

		return new LinkedHashMap<>();
	}
```

#### Secret 

- SecretsPropertySourceLocator#locate

```java
	@Override
	public PropertySource locate(Environment environment) {
		if (environment instanceof ConfigurableEnvironment) {
			ConfigurableEnvironment env = (ConfigurableEnvironment) environment;

			List<SecretsConfigProperties.NormalizedSource> sources = this.properties.determineSources();
			CompositePropertySource composite = new CompositePropertySource("composite-secrets");
			if (this.properties.isEnableApi()) {
				// 获取Secret
				sources.forEach(s -> composite.addFirstPropertySource(
					getKubernetesPropertySourceForSingleSecret(env, s)));
			}

			// read for secrets mount
			// 读取并加入到容器中
			putPathConfig(composite);

			return composite;
		}
		return null;
	}
```

- SecretsPropertySourceLocator#getKubernetesPropertySourceForSingleSecret

```java
	private MapPropertySource getKubernetesPropertySourceForSingleSecret(
		ConfigurableEnvironment environment,
		SecretsConfigProperties.NormalizedSource normalizedSource) {

		String configurationTarget = this.properties.getConfigurationTarget();
		// 加载 Secret 属性 source
		return new SecretsPropertySource(this.client,
			environment,
			getApplicationName(environment, normalizedSource.getName(), configurationTarget),
			getApplicationNamespace(this.client, normalizedSource.getNamespace(), configurationTarget),
			normalizedSource.getLabels());
	}
```

- SecretsPropertySource

```java
	public SecretsPropertySource(KubernetesClient client, Environment env, String name,
	                             String namespace, Map<String, String> labels) {
		super(getSourceName(client, env, name, namespace),
			getSourceData(client, env, name, namespace, labels));
	}
```

- SecretsPropertySource#getSourceData

获取 Secret 的流程和获取 ConfigMap 一样，不同的是 Secret 在放入环境中之前，需要先通过 Base64 解码

```java
	private static Map<String, Object> getSourceData(KubernetesClient client,
	                                                 Environment env, String name, String namespace, Map<String, String> labels) {
		Map<String, Object> result = new HashMap<>();

		try {
			// Read for secrets api (named)
			// 根据名称和命名空间获取 secret
			Secret secret;
			if (StringUtils.isEmpty(namespace)) {
				secret = client.secrets()
				               .withName(name)
				               .get();
			} else {
				secret = client.secrets()
				               .inNamespace(namespace)
				               .withName(name)
				               .get();
			}
			// 解码
			putAll(secret, result);

			// Read for secrets api (label)
			// 根据 label 读取 Secret
			if (!labels.isEmpty()) {
				if (StringUtils.isEmpty(namespace)) {
					client.secrets()
					      .withLabels(labels)
					      .list()
					      .getItems()
					      .forEach(s -> putAll(s, result));
				} else {
					client.secrets()
					      .inNamespace(namespace)
					      .withLabels(labels)
					      .list()
					      .getItems()
					      .forEach(s -> putAll(s, result));
				}
			}
		} catch (Exception e) {
			LOG.warn("Can't read secret with name: [" + name + "] or labels [" + labels
				+ "] in namespace:[" + namespace + "] (cause: " + e.getMessage()
				+ "). Ignoring");
		}

		return result;
	}
```

## 监听配置

支持两种方式的监听配置，一种是通过和 Kubernetes 建立长连接，当配置发生变化时可以立即推送，另一种是通过长轮询的方式，通过定时任务来实现

配置的监听必须显式开启

### Bean 初始化 

Bean 的初始化是在 `org.springframework.cloud.kubernetes.config.reload.ConfigReloadAutoConfiguration.ConfigReloadAutoConfigurationBeans` 中实现的

- 监听配置变化

根据配置选择是通过轮询还是监听事件方式实现，默认是监听事件

```java
		@Bean
		@ConditionalOnMissingBean
		public ConfigurationChangeDetector propertyChangeWatcher(ConfigReloadProperties properties, ConfigurationUpdateStrategy strategy) {
			switch (properties.getMode()) {
				case POLLING:
					return new PollingConfigurationChangeDetector(this.environment,
						properties,
						this.kubernetesClient,
						strategy,
						this.configMapPropertySourceLocator,
						this.secretsPropertySourceLocator);
				case EVENT:
					return new EventBasedConfigurationChangeDetector(this.environment,
						properties,
						this.kubernetesClient,
						strategy,
						this.configMapPropertySourceLocator,
						this.secretsPropertySourceLocator);
			}
			throw new IllegalStateException("Unsupported configuration reload mode: " + properties.getMode());
		}
```

- 配置更新策略

配置更新支持三种策略，分别是重启，刷新，和关闭，关闭应用依赖于健康检查，当发现应用被关闭后需要通过 Kubernetes 主动拉起

默认策略是刷新上下文

```java
		@Bean
		@ConditionalOnMissingBean
		public ConfigurationUpdateStrategy configurationUpdateStrategy(
			ConfigReloadProperties properties,
			ConfigurableApplicationContext ctx,
			@Autowired(required = false) RestartEndpoint restarter,
			ContextRefresher refresher) {
			switch (properties.getStrategy()) {
				// 重启
				case RESTART_CONTEXT:
					Assert.notNull(restarter, "Restart endpoint is not enabled");
					return new ConfigurationUpdateStrategy(properties.getStrategy().name(),
						() -> {
							wait(properties);
							restarter.restart();
						});
				//	刷新
				case REFRESH:
					return new ConfigurationUpdateStrategy(properties.getStrategy().name(),
						refresher::refresh);
				//	关闭
				case SHUTDOWN:
					return new ConfigurationUpdateStrategy(properties.getStrategy().name(),
						() -> {
							wait(properties);
							ctx.close();
						});
			}
			throw new IllegalStateException("Unsupported configuration update strategy: "
				+ properties.getStrategy());
		}
```

### 监听实现

![spring-cloud-kubernetes-configuration-change-detector.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/spring-cloud-kubernetes-configuration-change-detector.png)

`PollingConfigurationChangeDetector` 和 `EventBasedConfigurationChangeDetector` 都是 `ConfigurationChangeDetector`的子类

#### polling

默认 15s 拉取一次配置

- PollingConfigurationChangeDetector#executeCycle

```java
	@Scheduled(initialDelayString = "${spring.cloud.kubernetes.reload.period:15000}",
		fixedDelayString = "${spring.cloud.kubernetes.reload.period:15000}")
	public void executeCycle() {

		boolean changedConfigMap = false;
		// 监控 ConfigMap
		if (this.properties.isMonitoringConfigMaps()) {
			List<? extends MapPropertySource> currentConfigMapSources = findPropertySources(ConfigMapPropertySource.class);

			if (!currentConfigMapSources.isEmpty()) {
				changedConfigMap = changed(
					locateMapPropertySources(this.configMapPropertySourceLocator, this.environment),
					currentConfigMapSources);
			}
		}

		boolean changedSecrets = false;
		//  监控 Secret
		if (this.properties.isMonitoringSecrets()) {
			List<MapPropertySource> currentSecretSources = locateMapPropertySources(this.secretsPropertySourceLocator, this.environment);

			if (currentSecretSources != null && !currentSecretSources.isEmpty()) {
				List<SecretsPropertySource> propertySources = findPropertySources(SecretsPropertySource.class);
				changedSecrets = changed(currentSecretSources, propertySources);
			}
		}

		// 当发生变化时，更新属性
		if (changedConfigMap || changedSecrets) {
			reloadProperties();
		}
	}

```

拉取配置，通过调用`change`方法进行比较，判断是否发生变化，如果发生变化，则调用 `reloadProperties` 方法刷新

- ConfigurationChangeDetector#reloadProperties

```java
	public void reloadProperties() {
		this.log.info("Reloading using strategy: " + this.strategy.getName());
		this.strategy.reload();
	}
```

最终调用配置策略的 reload 方法，重新加载配置，需要注意的是，要被刷新的属性类应当通过 `@RefreshScope`或 `@ConfigurationProperties`注解修饰，这样才能监听到上下文的变化

#### Event

- EventBasedConfigurationChangeDetector#watch 

```java
	@PostConstruct
	public void watch() {
		boolean activated = false;

		// 如果监控 ConfigMap
		if (this.properties.isMonitoringConfigMaps()) {
			try {
				String name = "config-maps-watch";
				// Watch 是通过 WebSocket 实现的
				this.watches.put(name, this.kubernetesClient.configMaps()
				                                            .watch(new Watcher<ConfigMap>() {
					                                            @Override
					                                            public void eventReceived(Action action,
					                                                                      ConfigMap configMap) {
						                                            onEvent(configMap);
					                                            }

					                                            @Override
					                                            public void onClose(KubernetesClientException e) {
					                                            }
				                                            }));
				activated = true;
				this.log.info("Added new Kubernetes watch: " + name);
			} catch (Exception e) {
				this.log.error(
					"Error while establishing a connection to watch config maps: configuration may remain stale",
					e);
			}
		}

		// 如果监控 Secret
		if (this.properties.isMonitoringSecrets()) {
			try {
				activated = false;
				String name = "secrets-watch";
				this.watches.put(name,
					this.kubernetesClient.secrets().watch(new Watcher<Secret>() {
						@Override
						public void eventReceived(Action action, Secret secret) {
							onEvent(secret);
						}

						@Override
						public void onClose(KubernetesClientException e) {
						}
					}));
				activated = true;
				this.log.info("Added new Kubernetes watch: " + name);
			} catch (Exception e) {
				this.log.error(
					"Error while establishing a connection to watch secrets: configuration may remain stale",
					e);
			}
		}

		if (activated) {
			this.log.info(
				"Kubernetes event-based configuration change detector activated");
		}
	}

```

当收到消息时，将会调用 `onEvent`方法处理事件

- EventBasedConfigurationChangeDetector#onEvent

```java
	private void onEvent(ConfigMap configMap) {
		// 加载配置，检测是否变化
		boolean changed = changed(
			locateMapPropertySources(this.configMapPropertySourceLocator, this.environment),
			findPropertySources(ConfigMapPropertySource.class)
		);
		// 如果变化了，则重新加载属性
		if (changed) {
			this.log.info("Detected change in config maps");
			reloadProperties();
		}
	}

	private void onEvent(Secret secret) {
		boolean changed = changed(
			locateMapPropertySources(this.secretsPropertySourceLocator, this.environment),
			findPropertySources(SecretsPropertySource.class));
		if (changed) {
			this.log.info("Detected change in secrets");
			reloadProperties();
		}
	}
```

重新加载配置，当发现配置发生变化时会调用 `reloadProperties`方法更新配置