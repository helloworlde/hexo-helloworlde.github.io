---
title: Spring Cloud 使用 Consul 作为配置中心
date: 2020-09-20 22:28:50
tags:
    - Java
    - SpringCloud
categories: 
    - Java
    - SpringCloud
---

# Spring Cloud 使用 Consul 作为配置中心

## 加载配置 

加载配置是通过 `ConsulPropertySourceLocator` 来实现的，该类是 `PropertySourceLocator`接口的实现类

### Bean 初始化

```java
		@Bean
		public ConsulPropertySourceLocator consulPropertySourceLocator(ConsulConfigProperties consulConfigProperties) {
			return new ConsulPropertySourceLocator(this.consul, consulConfigProperties);
		}
```

### 获取配置

获取配置是通过 PropertySourceLocator#locate 方法实现的，最终将获取到属性添加到环境中

- ConsulPropertySourceLocator#locate

获取配置时，根据应用名称，路径，环境及配置类型拼接相应的路径，然后调用 Consul 获取 KV 值的接口，获取相应的配置，根据类型解析后放入环境中

```java
	@Override
	@Retryable(interceptor = "consulRetryInterceptor")
	public PropertySource<?> locate(Environment environment) {
		if (environment instanceof ConfigurableEnvironment) {
			ConfigurableEnvironment env = (ConfigurableEnvironment) environment;

			String appName = this.properties.getName();

			if (appName == null) {
				appName = env.getProperty("spring.application.name");
			}

			List<String> profiles = Arrays.asList(env.getActiveProfiles());

			String prefix = this.properties.getPrefix();

			List<String> suffixes = new ArrayList<>();
			// 不是文件类型的时候，后缀为 /，否则就是配置文件的后缀
			if (this.properties.getFormat() != FILES) {
				suffixes.add("/");
			} else {
				suffixes.add(".yml");
				suffixes.add(".yaml");
				suffixes.add(".properties");
			}

			// 路径
			String defaultContext = getContext(prefix, this.properties.getDefaultContext());

			for (String suffix : suffixes) {
				this.contexts.add(defaultContext + suffix);
			}
			// 追加环境及文件类型
			for (String suffix : suffixes) {
				addProfiles(this.contexts, defaultContext, profiles, suffix);
			}

			String baseContext = getContext(prefix, appName);

			// 应用名称前缀
			for (String suffix : suffixes) {
				this.contexts.add(baseContext + suffix);
			}
			for (String suffix : suffixes) {
				addProfiles(this.contexts, baseContext, profiles, suffix);
			}

			Collections.reverse(this.contexts);

			CompositePropertySource composite = new CompositePropertySource("consul");

			for (String propertySourceContext : this.contexts) {
				try {
					ConsulPropertySource propertySource = null;
					if (this.properties.getFormat() == FILES) {
						// 获取值
						Response<GetValue> response = this.consul.getKVValue(propertySourceContext, this.properties.getAclToken());
						// 添加当前索引
						addIndex(propertySourceContext, response.getConsulIndex());
						// 如果值不为空，则更新值并初始化
						if (response.getValue() != null) {
							ConsulFilesPropertySource filesPropertySource = new ConsulFilesPropertySource(propertySourceContext, this.consul, this.properties);
							// 解析配置内容
							filesPropertySource.init(response.getValue());
							propertySource = filesPropertySource;
						}
					} else {
						propertySource = create(propertySourceContext, this.contextIndex);
					}
					if (propertySource != null) {
						composite.addPropertySource(propertySource);
					}
				} catch (Exception e) {
					if (this.properties.isFailFast()) {
						log.error("Fail fast is set and there was an error reading configuration from consul.");
						ReflectionUtils.rethrowRuntimeException(e);
					} else {
						log.warn("Unable to load consul config from " + propertySourceContext, e);
					}
				}
			}

			return composite;
		}
		return null;
	}
```



## 监听配置

Consul 监听配置是通过定时任务实现的，

### Bean 初始化 

Bean 的初始化是在 `org.springframework.cloud.consul.config.ConsulConfigAutoConfiguration` 中实现的

```java
		@Bean
		@ConditionalOnProperty(name = "spring.cloud.consul.config.watch.enabled", matchIfMissing = true)
		public ConfigWatch configWatch(ConsulConfigProperties properties,
		                               ConsulPropertySourceLocator locator,
		                               ConsulClient consul,
		                               @Qualifier(CONFIG_WATCH_TASK_SCHEDULER_NAME) TaskScheduler taskScheduler) {
			return new ConfigWatch(properties, consul, locator.getContextIndexes(), taskScheduler);
		}
```

### 监听实现

ConfigWatch 类实现了 `ApplicationEventPublisherAware` 和 `SmartLifecycle` 接口，

- 启动

当应用启动后，会调用 SmartLifecycle 的 start 方法，然后初始化配置监听，通过向线程池添加一个定时任务，实现配置的定时拉取，定时任务默认周期是 1s

```java
	@Override
	public void start() {
		if (this.running.compareAndSet(false, true)) {
			this.watchFuture = this.taskScheduler.scheduleWithFixedDelay(
				this::watchConfigKeyValues, this.properties.getWatch().getDelay());
		}
	}
```

- 监听

监听时会遍历所有的key，根据 key 从 Consul 获取相应的数据，判断 Index 是否发生变化，如果发生变化，则发送 `RefreshEvent` 事件，需要手动实现事件监听以响应配置bai

```java
	// Timed 是 Prometheus 的监控
	@Timed("consul.watch-config-keys")
	public void watchConfigKeyValues() {
		if (this.running.get()) {
			// 遍历所有的配置的 key
			for (String context : this.consulIndexes.keySet()) {

				// turn the context into a Consul folder path (unless our config format
				// are FILES)
				if (this.properties.getFormat() != FILES && !context.endsWith("/")) {
					context = context + "/";
				}

				// 根据配置返回的 index 判断是否发生变化
				try {
					Long currentIndex = this.consulIndexes.get(context);
					if (currentIndex == null) {
						currentIndex = -1L;
					}

					log.trace("watching consul for context '" + context + "' with index " + currentIndex);

					// use the consul ACL token if found
					String aclToken = this.properties.getAclToken();
					if (StringUtils.isEmpty(aclToken)) {
						aclToken = null;
					}

					// 获取指定的 key
					Response<List<GetValue>> response = this.consul.getKVValues(context, aclToken, new QueryParams(this.properties.getWatch().getWaitTime(), currentIndex));

					// if response.value == null, response was a 404, otherwise it was a
					// 200
					// reducing churn if there wasn't anything
					if (response.getValue() != null && !response.getValue().isEmpty()) {
						Long newIndex = response.getConsulIndex();

						// 判断 key 的 index 是否相等，如果发生变化，则发出 RefreshEvent 事件
						if (newIndex != null && !newIndex.equals(currentIndex)) {
							// don't publish the same index again, don't publish the first
							// time (-1) so index can be primed
							// 没有发布过这个 index 的事件，且不是第一次发布
							if (!this.consulIndexes.containsValue(newIndex) && !currentIndex.equals(-1L)) {
								log.trace("Context " + context + " has new index " + newIndex);
								// 发送事件
								RefreshEventData data = new RefreshEventData(context, currentIndex, newIndex);
								this.publisher.publishEvent(new RefreshEvent(this, data, data.toString()));
							} else if (log.isTraceEnabled()) {
								log.trace("Event for index already published for context " + context);
							}
							this.consulIndexes.put(context, newIndex);
						} else if (log.isTraceEnabled()) {
							log.trace("Same index for context " + context);
						}
					} else if (log.isTraceEnabled()) {
						log.trace("No value for context " + context);
					}

				} catch (Exception e) {
					// only fail fast on the initial query, otherwise just log the error
					if (this.firstTime && this.properties.isFailFast()) {
						log.error("Fail fast is set and there was an error reading configuration from consul.");
						ReflectionUtils.rethrowRuntimeException(e);
					} else if (log.isTraceEnabled()) {
						log.trace("Error querying consul Key/Values for context '" + context + "'", e);
					} else if (log.isWarnEnabled()) {
						// simplified one line log message in the event of an agent
						// failure
						log.warn("Error querying consul Key/Values for context '" + context + "'. Message: " + e.getMessage());
					}
				}
			}
		}
		this.firstTime = false;
	}
```