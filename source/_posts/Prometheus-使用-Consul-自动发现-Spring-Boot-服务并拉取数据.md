---
title: Prometheus 使用 Consul 自动发现 Spring Boot 服务并拉取数据
date: 2020-05-16 14:49:38
tags:
    - Prometheus
    - Grafana
    - Consul
    - SpringBoot
categories: 
    - Prometheus
    - Grafana
    - Consul
    - SpringBoot
---

# Prometheus 使用 Consul 自动发现 Spring Boot 服务并拉取数据

使用 Prometheus监控 SpringBoot 应用，当应用很多，且上下线频繁时，需要不断的更改 Prometheus 的配置文件，不能灵活的使用，可以通过为 Prometheus配置注册中心，从注册中心拉取应用数据获取监控数据

## 启动 Prometheus 

- 添加配置文件 prometheus.yaml

```bash
mkdir -p ~/docker/prometheus/config
vi ~/docker/prometheus/config/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
  - static_configs:
    - targets: []
    scheme: http
    timeout: 10s
    api_version: v1
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - localhost:9090
```

- 启动容器 

```bash
docker run \
    -d \
    --name prometheus \
    -p 9090:9090 \
    -v ~/docker/prometheus/config/prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus    
```

## 启动 Consul 

```bash
docker run \
       -d \
       --name consul \
       -p 8500:8500 \
       consul
```

## 启动 Spring Boot 应用 

### 添加监控 

- 添加依赖 build.gradle 

```groovy
    compile 'org.springframework.cloud:spring-cloud-starter-consul-discovery'
    compile 'org.springframework.boot:spring-boot-starter-actuator'
    compile 'io.micrometer:micrometer-core:1.5.1'
    compile 'io.micrometer:micrometer-registry-prometheus:1.5.1' 
```

- 修改应用配置 applicaiton.properties 

```
spring.cloud.consul.host=localhost
spring.cloud.consul.port=8500
spring.cloud.consul.discovery.service-name=${spring.application.name}
spring.cloud.consul.discovery.prefer-ip-address=true
spring.cloud.consul.discovery.health-check-url=http://host.docker.internal:${server.port}/actuator/health
spring.cloud.consul.discovery.tags=metrics=true

management.endpoints.web.exposure.include=*
# prometheus
management.metrics.tags.application=${spring.application.name}
```

上面配置中指定健康检查是为了 Consul 从容器访问宿主机的应用，指定tag是为了Prometheus 从Consul列表中拉取需要监控的指定应用


## 使用 Consul 发现服务

### 修改 Prometheus 配置

- prometheus.yaml

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
    - static_configs:
        - targets: []
      scheme: http
      timeout: 10s
      api_version: v1
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "consul"
    consul_sd_configs:
      - server: "host.docker.internal:8500"
    relabel_configs:
      - source_labels: ["__meta_consul_dc"]
        target_label: "dc"
      - source_labels: [__meta_consul_service]
        separator: ;
        regex: (.*)
        target_label: application
        replacement: $1
        action: replace
      - source_labels: [__address__]
        separator: ":"
        regex: (127.0.0.1):(.*)
        target_label: __address__
        replacement: host.docker.internal:${2}
        action: replace
      - source_labels: [__metrics_path__]
        separator: ;
        regex: /metrics
        target_label: __metrics_path__
        replacement: /actuator/prometheus
        action: replace
      - source_labels: ['__meta_consul_tags']
        regex: '^.*,metrics=true,.*$'
        action: keep
```

其中 :
- `consul_sd_configs`指定 Consul 的地址
- `relabel_configs` 指定配置标签覆盖规则
- `__meta_consul_service`
```yaml
      - source_labels: [__meta_consul_service]
        separator: ;
        regex: (.*)
        target_label: application
        replacement: $1
        action: replace
```
这个配置是将 `__meta_consul_service` 重新映射为 `application`字段，方便 Prometheus 查询

- `__address__`

```yaml
      - source_labels: [__address__]
        separator: ":"
        regex: (127.0.0.1):(.*)
        target_label: __address__
        replacement: host.docker.internal:${2}
        action: replace
```
这个配置是为了将 `127.0.0.1`的地址改为`host.docker.internal`，方便 Prometheus 从容器中访问宿主机的应用

- `__metrics_path__`

```yaml
      - source_labels: [__metrics_path__]
        separator: ;
        regex: /metrics
        target_label: __metrics_path__
        replacement: /actuator/prometheus
        action: replace
```

这个配置是为了将 Prometheus 默认的拉取数据 `/metrics`改成 `/actuator/prometheus`，方便从 Spring拉取，如果有多种不同的应用，数据路径不一样，建议设置多个job，以使用不同的规则


- `__meta_consul_tags`

```yaml
      - source_labels: ['__meta_consul_tags']
        regex: '^.*,metrics=true,.*$'
        action: keep
```
这个配置是为了筛选指定tag的应用，只有有这个tag的应用才会拉取监控数据，这里是 `metrics=true`，是在 Spring Boot的配置文件中配置的，这样就可以避免拉取不相关的数据的应用(如 Consul自己的数据，替换路径为`/actuator/prometheus`后无法获取到监控数据)

配置完成后重启 Prometheus

### 获取应用

- Consul 中注册的应用 

![prometheus-consul-consul-service-list.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/prometheus-consul-consul-service-list.png)

- Prometheus 的 Service Discovery

可以看到，consul这个服务下发现了三个服务，但是只有一个在拉取数据，另外两个被丢弃了，这是因为限制了只拉取 tag 为 `metrics=true`

![prometheus-consul-prometheus-discovery-list.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/prometheus-consul-prometheus-discovery-list.png)


- Prometheus 的 Target

可以看到 Target 也只拉取了这一个应用

![prometheus-consul-prometheus-target-list.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/prometheus-consul-prometheus-target-list.png)

- 查询监控数据

在 Prometheus 的 Graph中查询应用信息，也只能看到一个

```
process_uptime_seconds
```

![prometheus-consul-prometheus-query-result1.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/prometheus-consul-prometheus-query-result1.png)

- 修改另一个 Spring Boot 应用，添加监控tag

再次查询 Prometheus 数据，可以看到另一个应用也开始拉取数据

![prometheus-consul-prometheus-query-result2.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/prometheus-consul-prometheus-query-result2.png)


----

- 相关项目可以参考 [grpc](https://github.com/helloworlde/SpringBootCollection/tree/master/grpc)