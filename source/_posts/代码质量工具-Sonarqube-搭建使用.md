---
title: 代码质量工具 Sonarqube 搭建使用
date: 2018-08-01 00:31:41
tags:
    - Sonarqube
    - Tool
categories: 
    - Sonarqube
    - Tool
---

# 代码质量工具 Sonarqube 搭建使用

> [Sonarqube](https://www.sonarqube.org/) 是一个代码质量管理平台，可以结合不同的测试工具，代码分析工具，持续集成工具等提供代码质量可是化和管理的工具

- [在线工具](https://sonarcloud.io)
- 截图
![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo1.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo1.png)
![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo2.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo2.png)
![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo3.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo3.png)
![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo4.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SonarqubeDemo4.png)


## 使用

### 启动容器

``` bash
docker run -d --name sonarqube \
    -p 9000:9000 -p 9092:9092 \
    -e SONARQUBE_JDBC_USERNAME=root \
    -e SONARQUBE_JDBC_PASSWORD=123456 \
    -e SONARQUBE_JDBC_URL=jdbc:mysql://localhost:3308/sonar\?useUnicode=true\&characterEncoding=utf8 \
    sonarqube
```

### 分析项目

- Java - Maven

```bash
mvn sonar:sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=d84cd047d5a4e149af1f4d614e28ed5183ef0c50
```

- Java - Gradle 
 - build.gradle

```gradle
plugins {
  id "org.sonarqube" version "2.6"
}
```

执行

```bash
./gradlew sonarqube \ 
  -Dsonar.host.url=http://localhost:9000 \ 
  -Dsonar.login=d84cd047d5a4e149af1f4d614e28ed5183ef0c50
```