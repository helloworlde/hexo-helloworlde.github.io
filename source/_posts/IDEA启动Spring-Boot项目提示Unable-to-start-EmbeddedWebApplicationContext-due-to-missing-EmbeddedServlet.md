---
title: >-
  IDEA启动Spring Boot项目提示Unable to start EmbeddedWebApplicationContext due to
  missing EmbeddedServlet...
date: 2018-01-01 01:03:07
tags:
    - Java
    - SpringBoot
    - Exception 
    - IDEA
categories: 
    - Java
    - SpringBoot
    - Exception
    - IDEA
---
> 导入一个`gradle` 的`Spring Boot`项目，在启动的时候先提示找不到`HttpServletRequest`这个包，错误如下：

```
Caused by: java.lang.ClassNotFoundException: javax.servlet.http.HttpServletRequest
    at java.net.URLClassLoader.findClass(URLClassLoader.java:381) ~[na:1.8.0_45]
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424) ~[na:1.8.0_45]
    at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331) ~[na:1.8.0_45]
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357) ~[na:1.8.0_45]
    ... 29 common frames omitted

```

> 但是相应的Java代码没有报错，所以单独找了`servlet-api.jar`导入，并将其添加到`Module`中，但是导入后出现另一个错误：

```
org.springframework.context.ApplicationContextException: Unable to start embedded container; nested exception is org.springframework.context.ApplicationContextException: Unable to start EmbeddedWebApplicationContext due to missing EmbeddedServletContainerFactory bean.
    at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.onRefresh(EmbeddedWebApplicationContext.java:137) ~[spring-boot-1.5.2.RELEASE.jar:1.5.2.RELEASE]
....
```

> 找了很久都没有找到解决的方法，但是使用`gradle bootrun`是可以正常启动运行的，在另外一台机子上也没有任何问题，所以认为项目本身没有任何问题，是在IDEA启动的过程中出现了问题导致的，



> 看到有一篇使用`Maven`也遇到该问题的帖子，对比了依赖：

```
dependencies {
    compile('org.mybatis.spring.boot:mybatis-spring-boot-starter:1.2.0')
    compile('org.springframework.boot:spring-boot-starter-web')
    runtime('mysql:mysql-connector-java')
    testCompile('org.springframework.boot:spring-boot-starter-test')

    // this is for generate war file
    providedRuntime('org.springframework.boot:spring-boot-starter-tomcat')
}
```

> 然后将`providedRuntime`改成了`runtime`，重新`build`启动，没有任何问题
> 该问题产生的原因很可能是因为IDEA在启动的过程中并没有像`Gradle`一样做完全的`build`，只是进行了热更新，没有将需要的Jar包编译到项目里

> **`providedCompile` 和 `providedRuntime`。虽然它们有各自的`compile` 和 `runtime` 配置，但这些配置有相同的作用域，只是它们不会添加到 `war/jar` 文件中**。