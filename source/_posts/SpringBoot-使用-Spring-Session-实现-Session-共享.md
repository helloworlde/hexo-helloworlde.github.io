---
title: SpringBoot-使用 Spring Session 实现 Session 共享
date: 2018-04-08 15:09:41
tags:   
    - Java
    - SpringBoot 
    - SpringSession
    - Session
categories: 
    - Java
    - SpringBoot
    - SpringSession
    - Session
---

# SpringBoot 使用 Spring Session 实现 Session 共享

## 通过 Redis 共享 

### 配置
- 配置并启动 Redis
- 添加依赖

```gradle
dependencies {
    compile('org.springframework.session:spring-session')
    compile('org.springframework.boot:spring-boot-starter-data-redis')
    compile('org.springframework.boot:spring-boot-starter-web')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

- 添加配置

```
spring.session.store-type=redis
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.database=0
spring.redis.password=123456
```

### 使用

- 启用 Redis Session 

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.session.data.redis.config.annotation.web.http.EnableRedisHttpSession;

@SpringBootApplication
@EnableRedisHttpSession
public class SessionApplication {

    public static void main(String[] args) {
        SpringApplication.run(SessionApplication.class, args);
    }
}
```

`@EnableRedisHttpSession` 也可以不写，在配置文件里配置  `spring.session.store-type=redis` 即可

- 添加 Controller 

```java

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpSession;

@RestController
public class SessionController {

    @RequestMapping("/")
    @ResponseBody
    public String root(HttpSession session) {
        return session.getId();
    }

}
```

访问 `http://localhost:8080`, 此时会显示 `Session ID`, 重新启动应用，再次访问 `http://localhost:8080`，此时 `Session ID` 应该和第一次访问一致，说明 `Session` 已被正确共享

-------------------------


## 通过 JDBC 共享 

### 配置

- 创建 Spring Session 所需要的表 

```sql
CREATE TABLE SPRING_SESSION (
    SESSION_ID CHAR(36) NOT NULL,
    CREATION_TIME BIGINT NOT NULL,
    LAST_ACCESS_TIME BIGINT NOT NULL,
    MAX_INACTIVE_INTERVAL INT NOT NULL,
    PRINCIPAL_NAME VARCHAR(100),
    CONSTRAINT SPRING_SESSION_PK PRIMARY KEY (SESSION_ID)
);

CREATE INDEX SPRING_SESSION_IX1 ON SPRING_SESSION (LAST_ACCESS_TIME);

CREATE TABLE SPRING_SESSION_ATTRIBUTES (
    SESSION_ID CHAR(36) NOT NULL,
    ATTRIBUTE_NAME VARCHAR(200) NOT NULL,
    ATTRIBUTE_BYTES BLOB NOT NULL,
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_PK PRIMARY KEY (SESSION_ID, ATTRIBUTE_NAME),
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_FK FOREIGN KEY (SESSION_ID) REFERENCES SPRING_SESSION(SESSION_ID) ON DELETE CASCADE
);

CREATE INDEX SPRING_SESSION_ATTRIBUTES_IX1 ON SPRING_SESSION_ATTRIBUTES (SESSION_ID);
```

该脚本位于 `org.springframework.session.jdbc.JdbcOperationsSessionRepository` 下，可以根据需要执行

- 添加依赖

```gradle
    compile('org.springframework.boot:spring-boot-starter-web')
    compile('org.springframework.session:spring-session')
    compile('org.mybatis.spring.boot:mybatis-spring-boot-starter:1.3.1')
    runtime('mysql:mysql-connector-java')
    //runtime('com.h2database:h2')
    testCompile('org.springframework.boot:spring-boot-starter-test')

```

- 添加配置

```
# DataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/session?useSSL=false
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.platform=mysql

#spring.datasource.driver-class-name=org.h2.Driver
#spring.datasource.url=jdbc:h2:mem:test
#spring.datasource.username=root
#spring.datasource.password=123456
#spring.datasource.platform=h2

spring.datasource.initialize=true
spring.datasource.continue-on-error=true

spring.session.store-type=jdbc

# MyBatis
#mybatis.type-aliases-package=cn.com.hellowood.session.dao
#mybatis.mapper-locations=mappers/**Mapper.xml

```

### 使用

- 启用 JDBC Session 

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.session.data.redis.config.annotation.web.http.EnableJdbcHttpSession;

@SpringBootApplication
@EnableJdbcHttpSession
public class SessionApplication {

    public static void main(String[] args) {
        SpringApplication.run(SessionApplication.class, args);
    }
}
```

`@EnableJdbcHttpSession` 也可以不写，在配置文件里配置  `spring.session.store-type=jdbc` 即可

- 添加 Controller 

```java

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpSession;

@RestController
public class SessionController {

    @RequestMapping("/")
    @ResponseBody
    public String root(HttpSession session) {
        return session.getId();
    }

}
```

访问 `http://localhost:8080`, 此时会显示 `Session ID`, 重新启动应用，再次访问 `http://localhost:8080`，此时 `Session ID` 应该和第一次访问一致，说明 `Session` 已被正确共享



