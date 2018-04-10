---
title: Spring Boot Admin 管理应用-客户端注册
date: 2018-01-01 12:15:48
tags:
    - Java
    - SpringBoot 
    - Actuator
categories: 
    - Java
    - SpringBoot
    - Actuator
---
# SpringBoot Admin管理应用-客户端注册
> 客户端注册可以通过直接注册到管理应用和通过注册中心注册两种方式

------------------

## 直接注册到管理应用
> 直接注册到管理应用只需要一个Server和一个Client就可以，Client配置admin Server地址就可以实现管理

#### 配置管理应用Server
#### 修改客户端应用

1. 添加依赖

```gradle
    compile('de.codecentric:spring-boot-admin-starter-client:1.5.4')
    compile('org.springframework.boot:spring-boot-starter-actuator')
```
 2.   修改配置文件(application.properties)，指明Server地址 
      
```
    spring.boot.admin.url=http://localhost:8080
```

---------------

## 通过注册中心注册
> 通过注册中心注册可以用于大量应用的管理，通过一个注册中心来管理注册，客户端和管理应用通过注册中心实现管理


> 使用Spring Cloud Eureka作为注册中心，需要一个Eureka服务应用，一个Admin Server应用和一个被管理的客户端应用

#### 配置Eureka
#### 配置Admin Server
#### 配置Client 
1. 添加依赖（build.gradle）
```gradle
    compile('org.springframework.boot:spring-boot-starter-actuator')
    compile('org.springframework.cloud:spring-cloud-starter-eureka:1.3.4.RELEASE')
    compile('org.jolokia:jolokia-core:1.3.7')
```
2. 添加启动配置项（bootstrap.properties）
```
info.version=1.0.0
spring.application.name=APPLICATION_NAME
# Eureka应用的URL
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```
3. 修改应用启动文件（Application.java）
```
@SpringBootApplication
@EnableEurekaClient //添加注册，向Eureka注册该应用
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```