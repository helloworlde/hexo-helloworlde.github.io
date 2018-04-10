---
title: Docker 构建 SpringBoot 应用
date: 2018-04-08 15:42:39
tags:
    - Docker
    - SpringBoot
    - Java
categories: 
    - Docker  
    - SpringBoot
    - Java
---

# 用 Docker 构建 SpringBoot 应用

- 启动 Docker，并生成 SpringBoot 应用

- 修改 `build.gradle` 文件

```
buildscript {
    ext {
        springBootVersion = '2.0.0.RELEASE'
    }
    repositories {
        maven { url "https://plugins.gradle.org/m2/" }
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("gradle.plugin.com.palantir.gradle.docker:gradle-docker:0.19.2")
    }
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'
apply plugin: 'com.palantir.docker'

group = 'cn.com.hellowood'
sourceCompatibility = 1.8
version = '1.0.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    compile('org.springframework.boot:spring-boot-starter-web')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}

docker {
    name "${project.group}/${jar.baseName}"
    files jar.archivePath
    buildArgs(['JAR_FILE': "${jar.archiveName}"])
}

```

- 添加 `Dockerfile`文件

```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG JAR_FILE
ADD ${JAR_FILE} app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

- 构建

```
gradle build docker
```

此时会生成`Docker`镜像

```
$ docker image ls 
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
cn.com.hellowood/docker   latest              94eefe321973        4 minutes ago       118MB

```

- 运行

```
docker run --name docker -p 8080:8080 cn.com.hellowood/docker 
```

此时可以在控制台看到应用的启动日志，项目启动之后可以访问`http://localhost:8080`,和本地启动应用访问一致

