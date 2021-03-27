---
title: SpringBoot中配置使用Druid数据库连接池
date: 2018-01-01 00:52:18
tags:
    - Java
    - SpringBoot 
    - Druid
categories: 
    - Java
    - SpringBoot
    - Druid
---
> Druid 是阿里巴巴开源的Java数据库连接池插件，Druid能够提供强大的监控和扩展功能

## 配置使用Druid数据库连接池
### 添加依赖
- Gradle

```
compile('com.alibaba:druid-spring-boot-starter:${version}')
```

- Maven

```
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>${version}</version>
```

### 配置数据库连接
> 数据库配置和默认配置基本一样，按照正常配置就可以，具体可以参考[https://github.com/alibaba/druid/blob/master/druid-spring-boot-starter/README.md](https://github.com/alibaba/druid/blob/master/druid-spring-boot-starter/README.md)

- JDBC 配置

```
spring.datasource.druid.url= # 或spring.datasource.url= 
spring.datasource.druid.username= # 或spring.datasource.username=
spring.datasource.druid.password= # 或spring.datasource.password=
spring.datasource.druid.driver-class-name= #或 spring.datasource.driver-class-name=
```

- 连接池配置

```
spring.datasource.druid.initial-size=
spring.datasource.druid.max-active=
spring.datasource.druid.min-idle=
spring.datasource.druid.max-wait=
spring.datasource.druid.pool-prepared-statements=
spring.datasource.druid.max-pool-prepared-statement-per-connection-size= 
spring.datasource.druid.max-open-prepared-statements= #和上面的等价
spring.datasource.druid.validation-query=
spring.datasource.druid.validation-query-timeout=
spring.datasource.druid.test-on-borrow=
spring.datasource.druid.test-on-return=
spring.datasource.druid.test-while-idle=
spring.datasource.druid.time-between-eviction-runs-millis=
spring.datasource.druid.min-evictable-idle-time-millis=
spring.datasource.druid.max-evictable-idle-time-millis=
spring.datasource.druid.filters= #配置多个英文逗号分隔
....//more
```

- 监控配置

```
# WebStatFilter配置，说明请参考Druid Wiki，配置_配置WebStatFilter
spring.datasource.druid.web-stat-filter.enabled= #是否启用StatFilter默认值true
spring.datasource.druid.web-stat-filter.url-pattern=
spring.datasource.druid.web-stat-filter.exclusions=
spring.datasource.druid.web-stat-filter.session-stat-enable=
spring.datasource.druid.web-stat-filter.session-stat-max-count=
spring.datasource.druid.web-stat-filter.principal-session-name=
spring.datasource.druid.web-stat-filter.principal-cookie-name=
spring.datasource.druid.web-stat-filter.profile-enable=

# StatViewServlet配置，说明请参考Druid Wiki，配置_StatViewServlet配置
spring.datasource.druid.stat-view-servlet.enabled= #是否启用StatViewServlet默认值true
spring.datasource.druid.stat-view-servlet.url-pattern=
spring.datasource.druid.stat-view-servlet.reset-enable=
spring.datasource.druid.stat-view-servlet.login-username=
spring.datasource.druid.stat-view-servlet.login-password=
spring.datasource.druid.stat-view-servlet.allow=
spring.datasource.druid.stat-view-servlet.deny=

# Spring监控配置，说明请参考Druid Github Wiki，配置_Druid和Spring关联监控配置
spring.datasource.druid.aop-patterns= # Spring监控AOP切入点，如x.y.z.service.*,配置多个英文逗号分隔
# 如果spring.datasource.druid.aop-patterns要代理的类没有定义interface请设置spring.aop.proxy-target-class=true
```

> 配置好了之后启动应用就可以正常使用了，查看统计信息可以访问 http://localhost:8080/druid/index.html


----------
## 参考模板

```
spring.datasource.druid.url=jdbc:mysql://localhost:3306/db?useSSL=false
spring.datasource.druid.slave.username=root
spring.datasource.druid.slave.password=123456

# Druid dataSource config
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.druid.initial-size=5
spring.datasource.druid.max-active=20
spring.datasource.druid.min-idle=5
spring.datasource.druid.max-wait=60000
spring.datasource.druid.pool-prepared-statements=true
spring.datasource.druid.max-pool-prepared-statement-per-connection-size=20
spring.datasource.druid.max-open-prepared-statements=20
spring.datasource.druid.validation-query=SELECT 1
spring.datasource.druid.validation-query-timeout=30000
spring.datasource.druid.test-on-borrow=true
spring.datasource.druid.test-on-return=false
spring.datasource.druid.test-while-idle=false
#spring.datasource.druid.time-between-eviction-runs-millis=
#spring.datasource.druid.min-evictable-idle-time-millis=
#spring.datasource.druid.max-evictable-idle-time-millis=10000

# Druid stat filter config
spring.datasource.druid.filters=stat,wall,log4j
spring.datasource.druid.web-stat-filter.enabled=true 
spring.datasource.druid.web-stat-filter.url-pattern=/druid/*
#spring.datasource.druid.web-stat-filter.exclusions=
spring.datasource.druid.web-stat-filter.session-stat-enable=true
spring.datasource.druid.web-stat-filter.session-stat-max-count=10
#spring.datasource.druid.web-stat-filter.principal-session-name=
#spring.datasource.druid.web-stat-filter.principal-cookie-name=
spring.datasource.druid.web-stat-filter.profile-enable=true
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*
spring.datasource.druid.filter.stat.db-type=mysql
spring.datasource.druid.filter.stat.log-slow-sql=true
spring.datasource.druid.filter.stat.slow-sql-millis=1000
spring.datasource.druid.filter.stat.merge-sql=true
spring.datasource.druid.filter.wall.enabled=true
spring.datasource.druid.filter.wall.db-type=mysql
spring.datasource.druid.filter.wall.config.delete-allow=true
spring.datasource.druid.filter.wall.config.drop-table-allow=false

# Druid manage page config
spring.datasource.druid.stat-view-servlet.enabled=true
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
spring.datasource.druid.stat-view-servlet.reset-enable=true
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=admin
#spring.datasource.druid.stat-view-servlet.allow=
#spring.datasource.druid.stat-view-servlet.deny=
```