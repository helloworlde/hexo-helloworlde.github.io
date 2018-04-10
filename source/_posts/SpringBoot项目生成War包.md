---
title: SpringBoot项目生成War包
date: 2018-01-01 00:50:49
tags:
    - Java
    - SpringBoot 
categories: 
    - Java
    - SpringBoot
---
> Spring Boot 项目默认生成Jar包，如果想发布到Tomcat还需要生成War包才能运行，SpringBoot官方文档中已经阐述了具体的操作方法，可以参考：[howto-create-a-deployable-war-file](http://docs.spring.io/spring-boot/docs/2.0.0.M3/reference/htmlsingle/#howto-create-a-deployable-war-file)；
> 以下使用Gradle作为Build工具

## 1. 向build.gradle文件添加依赖

- build.gradle
```
buildscript {
    ext {
        springBootVersion = '1.5.4.RELEASE'
    }
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

apply plugin: 'java'
apply plugin: 'war' // 生成war包
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'

version = ''
sourceCompatibility = 1.8

repositories {
    maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
}

dependencies {
    compile('org.springframework.boot:spring-boot-starter-data-redis')
    compile('org.mybatis.spring.boot:mybatis-spring-boot-starter:1.3.0')
    compile('org.springframework.boot:spring-boot-starter-web')
    compile('org.springframework.boot:spring-boot-starter-aop')
    compile('org.springframework.boot:spring-boot-starter-actuator')
    runtime('com.h2database:h2')
    runtime('mysql:mysql-connector-java')
    testCompile('org.springframework.boot:spring-boot-starter-test')

    // 生成war包
    providedRuntime ('org.springframework.boot:spring-boot-starter-tomcat')
}
```

## 2. 修改Application.java文件

- Application.java
```
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
}
```
> 也可以单独写在另一个类中，效果是一样的

- ServletInitializer.java

```
public class ServletInitializer extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
}

```

> 这样使用`gradle build`进行编译时就可以生成War包了
> 需要注意的是providedRuntime 仅在打包时才会添加依赖，可能会影响测试

> - War 插件添加了两个依赖配置： providedCompile 和 providedRuntime。虽然它们有各自的compile 和 runtime 配置，但这些配置有相同的作用域，只是它们不会添加到 WAR 文件中。要特别注意，这些 provided 配置的传递使用。假设你添加 commons-httpclient:commons-httpclient:3.0 依赖到任何一个 provided 配置。这个依赖又依赖于 commons-codec。这意味着 httpclient 和 commons-codec 都不会添加到你的 WAR 中，即使 commons-codec 是 compile 配置上的一个显示依赖。如果你不想要这种传递行为，只是把 provided 依赖声明成和commons-httpclient:commons-httpclient:3.0@jar 一样。