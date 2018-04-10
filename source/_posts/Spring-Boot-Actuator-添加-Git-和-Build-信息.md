---
title: Spring Boot Actuator 添加 Git 和 Build 信息
date: 2018-01-01 11:20:12
tags:
    - Java
    - SpringBoot 
    - Actuator
categories: 
    - Java
    - SpringBoot
    - Actuator
---
> 在使用 Spring Boot Actuator 时可以通过生成 Git 和编译文件来提供信息

## 添加 Git 信息
- 添加插件
> 在 build.gradle 文件中添加一下配置

```
buildscript {
 
    repositories {
        maven { url "https://plugins.gradle.org/m2/" }
    }

    dependencies {
        classpath("gradle.plugin.com.gorylenko.gradle-git-properties:gradle-git-properties:1.4.17")
    }
}

apply plugin: 'com.gorylenko.gradle-git-properties'

```
> 这样就会在 ` build\main\resource\`下生成 `git.properties`文件，该文件内会包含 Git 仓库的信息

- 其他配置
> build.gradle
```

gitProperties {
    // 日期格式
    dateFormat = "yyyy-MM-dd' 'HH:mm:ss"
    // 时区
    dateFormatTimeZone = "PST"
    // 生成的 git.properties 文件位置
    gitPropertiesDir = new File("${project.rootDir}/build/resources/main/")
    // git 文件所在目录
    gitRepositoryRoot = new File("${project.rootDir}/")
}

```
-------------------------------------
## 添加编译信息
- 添加配置信息
> 在 build.gradle 中添加

```
springBoot {
    buildInfo {
        // 自定义内容
        additionalProperties = [
                'name'   : 'RedisAPI',
                'version': '0.0.1'
        ]
    }
}

```
> 会在 `build\resources\MATE-INF\` 下生成 `build-info.properties` 文件

-------------------------------------

- 访问 `/info`

```
{
    "app": {
        "java": {
            "source": "1.8",
            "target": "1.8"
        },
        "encoding": "UTF-8"
    },
    "version": "Spring Boot application",
    "git": {
        "commit": {
            "message": {
                "full": ":sparkles: Add SpringBoot Admin Server monitor function",
                "short": ":sparkles: Add SpringBoot Admin Server monitor function"
            },
            "time": "2017-09-05T10:57-0700",
            "id": "3bb077a0710e31c2dfc345afdcdb52b9b5846f61",
            "id.abbrev": "3bb077a",
            "user": {
                "email": "hellowoodes@outlook.com",
                "name": "HelloWood"
            }
        },
        "branch": "master"
    },
    "build": {
        "version": "0.0.1",
        "artifact": "redis",
        "name": "RedisAPI",
        "group": "",
        "time": 1513519543000
    }
}
```

-------------------------------------

> 参考[https://docs.spring.io/spring-boot/docs/current/reference/html/howto-build.html#howto-build-info](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-build.html#howto-build-info)