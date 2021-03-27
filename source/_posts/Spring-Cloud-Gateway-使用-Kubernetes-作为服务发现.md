---
title: Spring Cloud Gateway 使用 Kubernetes 作为服务发现
date: 2020-09-20 22:25:04
tags:
    - Java
    - SpringCloud
categories: 
    - Java
    - SpringCloud
---

# Spring Cloud Gateway 使用 Kubernetes 作为服务发现

Spring Cloud Gateway 作为网关，通过用于执行一些通用逻辑后做请求转发，后端可能涉及到多个服务，每个服务又有多个实例，调用服务实例就需要动态的更新，可以通过注册中心来实现，如果部署在K8S集群中，可以直接使用K8S实现服务发现

## 应用

### Gateway 

#### 添加依赖

- build.gradle 

```groovy
plugins {
    id 'org.springframework.boot' version '2.2.6.RELEASE'
    id 'io.spring.dependency-management' version '1.0.9.RELEASE'
    id 'java'
}

ext {
    set('springCloudVersion', "Hoxton.SR1")
    set('springKubernetesVersion', "1.1.2.RELEASE")
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-kubernetes-discovery'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
   
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        mavenBom "org.springframework.cloud:spring-cloud-kubernetes-dependencies:${springKubernetesVersion}"
    }
}
```

- 如果是需要客户端实现负载均衡，依赖是

```groovy
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-kubernetes-all'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-ribbon'
```

#### 添加路由配置 

- bootstrap.yaml

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        catalogServicesWatchDelay: 5000
      client:
        master-url: https://kubernetes.docker.internal:6443
        ca-cert-data: xxx
        namespace: default
```

- applicaiton.yaml

```yaml
spring:
  application.name: gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          url-expression: "'http://'+serviceId+':'+port"
          lower-case-service-id: true
          
management:
  endpoints:
    web:
      exposure:
        include: '*'          
```

配置 `url-expression` 目的是为了在转发的时候直接转发到 Kubernetes 中相应的 Service 上去，默认的表达式为 `"'lb://'+serviceId"`，这种适用于通过 Consul 或者 Eureka，最终是根据服务的IP和端口访问，`spring-cloud-kubernetes`没有实现`com.netflix.loadbalancer.AbstractServerList`，所以不会进行IP转换，最终是通过服务名称查找Service 实现调用，所以不需要负载均衡

- 如果是客户端实现负载均衡，则不需要指定`"'http://'+serviceId+':'+port"`

### 部署

通过 Gateway 调用后端的 API 应用，该应用提供 REST 接口

#### 部署后端API应用 

- api.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: default
  labels:
    app: api
spec:
  type: ClusterIP
  ports:
    - port: 8080
  selector:
    app: api

---

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: api
  name: api
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: registry.cn-qingdao.aliyuncs.com/hellowoodes/api:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
```

- 部署 API 服务

```bash
kubectl apply -f api.yaml
```

#### 部署 Gateway 

- gateway.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: default
  labels:
    app: gateway
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30080
  selector:
    app: gateway

---

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: gateway
  name: gateway
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
        - name: gateway
          image: registry.cn-qingdao.aliyuncs.com/hellowoodes/api-gateway:1.0.6
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
```

- 部署 gateway 服务

```bash
kubectl apply -f gateway.yaml
```


### 测试

- 查询所有路由

```bash
http get localhost:30080/actuator/gateway/routes
```

- 请求 API 服务接口 

```bash
http get localhost:30080/api/hello
```

这里会根据 `api`这个path 查找名为 `api`的 Service，然后调用 `http://api:8080/hello`，这个接口返回hostname，也就是pod的名称

- 负载均衡 

启动多个 API 服务实例 

```bash
kubectl scale --replicas=2 deploy/api
```
待服务启动后多次调用`http://localhost:30080/api/hello`，返回的pod名称会变化，说明负载均衡生效


##  参考

- [how-to-set-up-spring-cloud-gateway-application-so-it-can-use-the-service-discove](https://stackoverflow.com/questions/56170511/how-to-set-up-spring-cloud-gateway-application-so-it-can-use-the-service-discove)
- [spring-cloud-kubernetes-spring-cloud-gateway-unable-to-find-instance-for-k8s](https://stackoverflow.com/questions/57594228/spring-cloud-kubernetes-spring-cloud-gateway-unable-to-find-instance-for-k8s)
- [salaboy/s1p_gateway](https://github.com/salaboy/s1p_gateway)
- [why use ribbon in k8s?How about clusterIp?](https://github.com/spring-cloud/spring-cloud-kubernetes/issues/541)
- [spring-cloud-starter-consul-discovery depends on spring-cloud-starter-netflix-ribbon](https://github.com/spring-cloud/spring-cloud-consul/issues/562)
- [What's causing: forbidden: User "system:anonymous" in some Cloud Providers](https://github.com/kubernetes-sigs/apiserver-builder-alpha/issues/225)