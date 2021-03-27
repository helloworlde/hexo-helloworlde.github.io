---
title: Spring MVC 使用 Spring Session 实现 Session 共享-Redis
date: 2018-01-01 00:07:20
tags:
    - Java
    - SrpingMVC 
    - Spring Session
categories: 
    - Java
    - SrpingMVC 
    - Spring Session
---
> 使用Spring Session 通过 Redis 实现 Session 共享，用于多实例应用

- Spring Session 官方文档[https://docs.spring.io/spring-session/docs/2.0.0.M4/reference/html5/#introduction](https://docs.spring.io/spring-session/docs/2.0.0.M4/reference/html5/#introduction)


----------
## Session 共享的简单原理
> 用户第一次访问应用时，应用会创建一个新的 Session，并且会将 Session 的 ID 作为 Cookie 缓存在浏览器，下一次访问时请求的头部中带着该 Cookie，应用通过获取的 Session ID 进行查找，如果该 Session 存在且有效，则继续该请求，如果 Cookie 无效或者 Session 无效，则会重新生成一个新的 Session
> 
> 在普通的 JavaEE 应用中，Session 信息放在内存中，当容器（如 Tomcat）关闭后，内存中的 Session 被销毁；重启后如果当前用户再去访问对应的是一个新的 Session ，在多实例中无法共享，一个用户只能访问指定的实例才能使用相同的 Session；
> 

> Session 共享实现的原理是将原来内存中的 Session 放在一个需要共享 Session 的实例都可以访问到的位置，如数据库，Redis 中等，从而实现多实例 Session 共享
> 
> 实现共享后，只要浏览器的 Cookie 中的 Session ID 没有改变，多个实例中的任意一个被销毁不会影响用户访问

----------
# Redis 方式实现
> 将 Session 对象序列化存储到 Redis 中，多个实例访问时都会使用该 Session，Spring Session 会管理 Session 信息的管理，无需其他操作

## 1. 添加依赖
- 在 pom.xml 文件里面添加如下依赖
```
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-redis</artifactId>
            <version>1.8.7.RELEAS</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session</artifactId>
            <version>1.3.1.RELEASE</version>
        </dependency>

```

## 2. 添加配置
- SpringConfig.xml 添加如下配置
```
    <!-- Spring Session共享 -->
    <bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration"/>
    <bean class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <property name="hostName" value="localhost"/>
        <property name="password" value="123456"/>
        <property name="port" value="6379"/>
        <property name="database" value="3"/>
    </bean>

```

## 3. 添加过滤器
- 在 web.xml 添加如下配置（过滤器）

```
    <filter>
        <filter-name>springSessionRepositoryFilter</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>springSessionRepositoryFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
```
> 该过滤器必须是第一个过滤器，所有的请求经过该过滤器后执行后续操作

------------------

- Session 信息
![这里写图片描述](http://img.blog.csdn.net/20170927194347149?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- Session 中的值以二进制信息保存在 Redis 中
![这里写图片描述](http://img.blog.csdn.net/20170927194422498?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)