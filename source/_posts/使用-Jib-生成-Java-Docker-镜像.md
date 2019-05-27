---
title: 使用 Jib 生成 Java Docker 镜像
date: 2018-07-16 00:17:43
tags:
    - Docker
    - Jib
    - Java
    - SpringBoot
categories: 
    - Docker
    - Jib
    - Java
    - SpringBoot
---

#  使用 Jib 生成  Java Docker 镜像

> [Jib](https://github.com/GoogleContainerTools/jib) 是谷歌最新开源的 Java 应用的 Docker 镜像生成工具，可以通过 Gradle 或 Maven 直接生成镜像并上传到仓库而不需要 Dockerfile 文件或者其他插件；Jib 支持将资源文件和类分层打包，可以大幅度提升生成镜像的速度

> 有一些其他的插件也可以通过 Docker 实现生成镜像，如[`com.palantir.docker`](https://helloworlde.github.io/2018/04/08/Docker-%E6%9E%84%E5%BB%BA-SpringBoot-%E5%BA%94%E7%94%A8/)等，但是都需要额外配置 Dockerfile, 如果应用仅需要通过 Dockerfile 构建镜像，建议使用 Jib 来提升构建和上传速度

## 使用

### [添加依赖](https://plugins.gradle.org/plugin/com.google.cloud.tools.jib)

- Gradle

```gradle

buildscript {
    repositories {
        maven { url 'https://plugins.gradle.org/m2/' }
    }
    dependencies {
        classpath "gradle.plugin.com.google.cloud.tools:jib-gradle-plugin:0.9.6"
    }
}

apply plugin: "com.google.cloud.tools.jib"

```

或

```gradle
plugins {
  id 'com.google.cloud.tools.jib' version '0.9.6'
}
```

### 构建镜像

> 以下方式都需要终端能够访问 `gcr.io`或 `hub.docker.com`等 Docker Hub 才能成功

- 直接构建并推送到 [GCR](https://cloud.google.com/container-registry/)
通过这种方式构建的镜像会以`gcr.io/distroless/java`为底层镜像，编译之后生成镜像，并推送到 Google 容器镜像仓库中：

```gradle
gradle jib 
```

- 指定推送的容器镜像仓库

[Google Container Center](https://cloud.google.com/container-registry/), [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/), [Docker Hub Registry](https://hub.docker.com/)之外的 Hub可能会失败，需要先登录到对应的 Docker Hub才可以

```gradle
gradle jib --image registry.hub.docker.com/helloworld/java:jib
```

或者在`build.gradle`中以下添加之后执行 `gradle jib` 

```
jib.to.image = 'registry.hub.docker.com/helloworld/java:jib'
```


- 保存在本地
需要本地 Docker 应用已经启动

```
gradle jibDockerBuild
```

- [自定义基础镜像和参数(build.gradle)](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#extended-usage)

```gradle
jib {
    from {
        image = 'registry.hub.docker.com/openjdk:8-jdk-alpine'
        auth {
            username = 'username'
            password = 'password'
        }
    }
    to {
        image = 'registry.hub.docker.com/helloword/java:jib'
        auth {
            username = 'username'
            password = 'password'
        }
        credHelper = 'osxkeychain'
    }
    container {
        jvmFlags = ['-Djava.security.egd=file:/dev/./urandom', '-Duser.timezone=GMT+08']
        mainClass = 'example.jib.MainClass'
        args = ['test]
        ports = ['8080']
    }
}
```

> - `from`：拉取的镜像的配置，默认为`gcr.io/distroless/java`
> - `to`:要生成的镜像的配置
> - `image`：拉取或生成的镜像名称
> - `auth`: 认证信息，分别为用户名和密码
> - [`credHelper`](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#authentication-methods)：鉴权信息的存放方式，Google 使用 `gcr`, AWS使用 `ecr-login`, DockerHub 根据平台使用 `osxkeychain`, `wincred`,`secretservice`,`pass`中的一种，可以参考 [docker-credential-helpers](https://github.com/docker/docker-credential-helpers)
> - `container`: 容器的属性
> - `jvmFlgs`: JVM 容器的参数，和 Dockerfile 的 `ENTRYPOINT`作用相同
> - `mainClass`: 启动类限定名
> - `args`: `main` 方法的传入参数
> - `ports`: 容器暴露的端口，和 Dockerfile 的`EXPOSE`作用相同

推荐`from` 改为 `registry.hub.docker.com/openjdk:8-jdk-alpine`, `to`改为 `registry.cn-qingdao.aliyuncs.com` 等国内的仓库，同时将  Docker 的镜像源改为[https://registry.docker-cn.com](https://registry.docker-cn.com ) 或 [https://docker.mirrors.ustc.edu.cn](https://docker.mirrors.ustc.edu.cn)以减少构建时间

---------------------------- 

## 问题

- 登录失败，未授权(401 Unauthorized)

出现这种问题一般是因为没有登录对应的Docker Registry 导致的，需要配置用户登录到相应的容器服务，或者在`from`和`to` 节点手动添加账户配置：

```gradle
        auth {
            username = 'username'
            password = 'password'
        }
```

- 连接超时( Connect to gcr.io/108.177.125.82:443 timed out)

这个问题很容易出现，主要是因为在国内无法访问[gcr.io](https://gcr.io)导致的，推荐使用 SS 的全局模式或者 [Outline](https://www.getoutline.org/en/home) 或其他可以访问到 [gcr.io](https://gcr.io) 的工具；一定要确保 Terminal 或者服务器可以访问到  [gcr.io](https://gcr.io) 才能在不指定仓库时构建成功；或者指定国内的仓库，不需要访问谷歌的仓库