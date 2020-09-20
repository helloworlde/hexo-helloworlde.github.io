---
title: Spring Cloud Gateway 使用 Kubernetes 实现负载均衡
date: 2020-09-20 22:23:18
tags:
    - Java
    - SpringCloud
categories: 
    - Java
    - SpringCloud
---

# Spring Cloud Gateway 使用 Kubernetes 实现负载均衡

在使用 Spring Cloud Gateway 作为服务服务发现时，可能会遇到 Gateway 并没有部署在服务所在的 Kubernetes 集群中，或者存在网络隔离，无法直接通过 Service Name 访问到相应的服务，这时候就需要通过 Service 的 IP 访问，但是，`spring-cloud-kubernetes-discovery `没有像 `spring-cloud-consul-discovery`一样实现服务负载均衡的接口，所以默认只会访问某个服务固定的 IP；这样就需要使用负载均衡的依赖

### 添加依赖

```groovy
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-kubernetes-all'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-ribbon'
```

`spring-cloud-starter-kubernetes-all`包含 Kubernetes 负载均衡的实现

### 添加配置 

- bootstrap.yaml

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        catalogServicesWatchDelay: 5000
      client:
        master-url: https://localhost:60002
        ca-cert-data: xxxx
        namespace: develop
```

- applicaiton.yaml

```yaml
spring:
  application:
    name: gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true

          lower-case-service-id: true

management:
  endpoints:
    web:
      exposure:
        include: '*'
```