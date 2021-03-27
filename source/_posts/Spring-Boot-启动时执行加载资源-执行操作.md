---
title: Spring Boot 启动时执行加载资源/执行操作
date: 2018-01-01 00:54:49
tags:
    - Java
    - SpringBoot 
categories: 
    - Java
    - SpringBoot
---


> SpringBoot 在启动的时候加载资源或者执行操作，进行初始化来执行特定操作，SpringBoot已经提供了这样的接口，通过实现该接口就可以实现需要的操作

------------------

## 实现CommandLineRunner接口
```
@Order(value=2)
@Component
public class CommandLineRunnerListenerImpl implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
       // 执行操作
    }
}
```
- 可以通过指定```@Order```的值来控制启动的顺序，值越小表示越先执行


## 实现ApplicationListener接口

```
@Component
public class CommandLineRunnerListenerImpl implements ApplicationListener {

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // 执行操作
    }
}
```