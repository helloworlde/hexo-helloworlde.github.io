---
title: SpringBoot 框架自带插件构建 Docker 镜像
date: 2020-09-20 22:31:43
tags:
    - SpringBoot 
    - Docker 
categories: 
    - SpringBoot 
    - Docker 
---

# SpringBoot 框架自带插件构建 Docker 镜像

Spring Boot 2.3.0 之后支持通过 buildpacks 插件构建 Docker 镜像，原理和执行过程与 Jib 类似，支持 Spring Boot 项目的分层构建，当代码改动后，只需更新代码部分，可以减少构建后 push 和 pull 镜像的时间，减少镜像存储的成本

底层是通过 [Buildpacks](https://buildpacks.io/) 构建，Buildpacks是 Dockerfile 的一个替代方案。Buildpacks 能够自动探测运行 Docker 容器中的应用时所需要的软件，例如，它会探测应用中所使用的 Java 版本，基于该版本，buildpack 会选择所指定的 JRE 并构建 Docker 镜像

## 优点

- Spring Boot 编译插件集成，不需要额外的配置
- JVM 参数计算，支持通过线程数量，类数量等，动态的计算适合的 JVM 参数
- 会在构建过程中加入 OOM Killer，当 OOM 时自动杀死应用

## 弊端

- 在镜像中指定了内存参数，CPU数量等
- 不能灵活添加自定义文件，只能添加应用中的文件为单独的层
- 构建镜像、运行镜像与 Buildpacks 强绑定，需要 Buildpacks 的参数才可以成功构建，如果要自定义信息，同样需要在基础镜像中加入 Buildpacks 的配置
- 不支持直接使用历史版本镜像的分层，当tag改变时会完全重新构建
- builder 和 runImage 中的内容是动态下载的，访问 GitHub 可能失败，也不安全

## 使用 

1. 更新 SpringBoot 版本为 2.3.2.RELEASE 
2. 启用分层

```groovy
bootJar {
    layered()
}    
```

3. 执行构建

会根据应用名称构建出一个镜像

```bash
gradle buildImage
```

4. 指定基础镜像、默认镜像以及镜像名称

```groovy
bootBuildImage {
    builder = "docker.io/hellowoodes/buildpacks-builer:base-platform-api-0.3"
    runImage = "docker.io/hellowoodes/buildpacks-run:base-cnb"
    imageName = "docker.io/hellowoodes/${artifactId}:${version}"
    environment = [
            "TZ"       : "Asia/Shanghai",
            "JAVA_OPTS": "-Djava.security.egd=file:/dev/./urandom"
    ]
}
```

## 自定义分层

目前只支持两种 layer，分别是 application 和 dependencies

- application 只支持将项目目录下的文件打包到镜像中，格式为文件目录
- dependencies 支持依赖，格式为 `groupId:artifactId:version`

共有4层，可以通过`java -Djarmode=layertools -jar image-0.0.1-SNAPSHOT.jar list` 查看：

- dependencies 正式版的依赖
- spring-boot-loader SpringBoot 启动文件
- snapshot-dependencies 快照版的依赖
- application 应用文件，即编译后的类，resource下面的文件


### 添加新的 layer

如需要将静态文件单独分为一个层，可以在 application 加入新的层的逻辑

需要注意的是，这个层的 layerOrder 得放到 application 层的前面，否则 application 层包含了所有的文件，这个层就找不到；因为文件的添加逻辑是先添加 include 中指定的，下一层如果没有指定则添加剩余的

暂时不支持将自定义的外部文件加入层中，需要自定义 buildpacks 才可以

```groovy
bootJar {
    layered {
        application {
            intoLayer("spring-boot-loader") {
                include "org/springframework/boot/loader/**"
            }
            // 自定义 static 层，存放静态文件
            intoLayer("static") {
                include "static/**"
            }
            intoLayer("application")
        }
        dependencies {
            intoLayer("snapshot-dependencies") {
                include "*:*:*SNAPSHOT"
            }
            intoLayer("dependencies")
        }
        layerOrder = ["dependencies", "spring-boot-loader", "snapshot-dependencies", "static", "application"]
    }
}
```

## 注意事项

- 默认的用户是 cnb
- 启动方式不同，是通过启动JarLauncher 来启动应用
- 自定义 Builder 镜像和 Stack 可以参考 [buildpacks/samples](https://github.com/buildpacks/samples)
- 构建镜像和基础运行镜像默认从 gcr.io 下载

## 参考文档

- [Packaging OCI Images](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/gradle-plugin/reference/html/#build-image)
- [java-buildpack-memory-calculator](https://github.com/cloudfoundry/java-buildpack-memory-calculator)
- [bell-sw/Liberica](https://github.com/bell-sw/Liberica)
- [HotSpot GC Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/considerations.html)
- [Java Buildpack Memory Calculator v3](https://docs.google.com/document/d/1vlXBiwRIjwiVcbvUGYMrxx2Aw1RVAtxq3iuZ3UK2vXA/edit#heading=h.uy41ishpv9zc) 
- [paketo-buildpacks/spring-boot](https://github.com/paketo-buildpacks/spring-boot)
- [Spring Boot Docker](https://spring.io/guides/topicals/spring-boot-docker)
- [Spring-Boot-2.3-Release-Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.3-Release-Notes)
- [buildpacks/samples](https://github.com/buildpacks/samples)
- [gradle-packaging-layers-configuration](https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/html/#packaging-layers-configuration)
- [maven-packaging-layers-configuration](https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/html/#repackage-layers-configuration)