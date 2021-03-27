---
title: Spring Cloud 监控服务器下 IP/URL 不正确导致无法注册的解决方法
date: 2018-01-01 11:51:23
tags:
    - Java
    - SpringBoot 
    - SpringCloud
    - Actuator
    - Issue
categories: 
    - Java
    - SpringBoot
    - SpringCloud
    - Actuator
    - Issue
---
> ## 本项目仅用到了 Spring Cloud，并没有使用 Eureka


> 在使用 Spring Cloud 对 Spring Boot 应用通过 Spring Admin 进行监控的时候，当 Admin Server 和被监控的应用都在本地启动的时候没有任何问题，但是当部署到 Server 上之后，Client 在注册到 Admin Server 上时 IP 地址不正确，发现是因为 Server 有内网和外网 IP，但是在应用注册的时候用了内网的 IP，Admin Server 访问该内网 IP 失败，所以应用无法注册

> 因为没有使用 Eureka，所以配置时需要用 Spring Cloud 的配置来处理

> 使用 Eureka 请参考 [http://www.jianshu.com/p/fa1e9c8e4f47](http://www.jianshu.com/p/fa1e9c8e4f47)

## 配置

- 修改配置文件，添加以下内容

```
spring.boot.admin.client.service-base-url=http://${your_ip}:${your_port}
```

-------------

## 说明

- 当没有任何配置的时候，会使用`http://bogon:9999/`注册
- 当 Client 加入了`spring.boot.admin.client.prefer-ip=true`的时候会以所得到的 IP 注册，此时 IP 为内网 IP，如果部署到服务器上将会无法注册
- 当 Client 配置为`spring.boot.admin.client.service-base-url=http://${your_ip}:${your_port}`时将会以所配置的地址进行注册