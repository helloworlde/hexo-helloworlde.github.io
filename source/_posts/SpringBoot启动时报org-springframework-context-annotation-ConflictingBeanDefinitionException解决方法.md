---
title: >-
  SpringBoot启动时报org.springframework.context.annotation.ConflictingBeanDefinitionException解决方法
date: 2018-01-01 00:53:44
tags:
    - Java
    - SpringBoot
    - Exception 
categories: 
    - Java
    - SpringBoot
    - Exception
---
> 在SpringBoot应用启动的过程中，因为报`org.springframework.context.annotation.ConflictingBeanDefinitionException`导致应用启动失败

----------------------

> **错误信息：**
```
Annotation-specified bean name 'personDubboConsumerService' for bean class 
[cn.com.hellowood.dubboclient.dubbo.PersonDubboConsumerService]
 conflicts with existing, non-compatible bean definition of same name and class 
 [cn.com.hellowood.dubbo.PersonDubboConsumerService]
```


> 这是因为在应用中使用到了多个该类的对象，而该类的对象通过注解的方式注入到应用中，在注入的过程中因为对象的名称重复导致了该异常
> **通过指定注入对象的名称可以解决这个问题**

- 原来的代码

```
@Component
public class PersonDubboConsumerService {

    @Reference(version = "1.0.0")
    PersonDubboService service;

    public void sayHello() {
        String name = "哈哈哈哈";
        Person person = service.sayHello(name);
    }
}
```
- 修改后（**Component注解加上名称就可以，要和另一个bean的名称不同**）

```
@Component("personConsumerService")
public class PersonDubboConsumerService {

    @Reference(version = "1.0.0")
    PersonDubboService service;

    public void sayHello() {
        String name = "哈哈哈哈";
        Person person = service.sayHello(name);
    }
}
```


----------
- 异常信息

```console
org.springframework.beans.factory.BeanDefinitionStoreException: Failed to parse configuration class [cn.com.hellowood.DubboClientApplication]; nested exception is org.springframework.context.annotation.ConflictingBeanDefinitionException: Annotation-specified bean name 'personDubboConsumerService' for bean class [cn.com.hellowood.dubboclient.dubbo.PersonDubboConsumerService] conflicts with existing, non-compatible bean definition of same name and class [cn.com.hellowood.dubbo.PersonDubboConsumerService]
    at org.springframework.context.annotation.ConfigurationClassParser.parse(ConfigurationClassParser.java:181) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ConfigurationClassPostProcessor.processConfigBeanDefinitions(ConfigurationClassPostProcessor.java:308) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(ConfigurationClassPostProcessor.java:228) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:270) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:93) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:687) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:525) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.refresh(EmbeddedWebApplicationContext.java:122) ~[spring-boot-1.5.6.RELEASE.jar:1.5.6.RELEASE]
    at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:693) [spring-boot-1.5.6.RELEASE.jar:1.5.6.RELEASE]
    at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:360) [spring-boot-1.5.6.RELEASE.jar:1.5.6.RELEASE]
    at org.springframework.boot.SpringApplication.run(SpringApplication.java:303) [spring-boot-1.5.6.RELEASE.jar:1.5.6.RELEASE]
    at org.springframework.boot.SpringApplication.run(SpringApplication.java:1118) [spring-boot-1.5.6.RELEASE.jar:1.5.6.RELEASE]
    at org.springframework.boot.SpringApplication.run(SpringApplication.java:1107) [spring-boot-1.5.6.RELEASE.jar:1.5.6.RELEASE]
    at cn.com.hellowood.DubboClientApplication.main(DubboClientApplication.java:14) [classes/:na]
Caused by: org.springframework.context.annotation.ConflictingBeanDefinitionException: Annotation-specified bean name 'personDubboConsumerService' for bean class [cn.com.hellowood.dubboclient.dubbo.PersonDubboConsumerService] conflicts with existing, non-compatible bean definition of same name and class [cn.com.hellowood.dubbo.PersonDubboConsumerService]
    at org.springframework.context.annotation.ClassPathBeanDefinitionScanner.checkCandidate(ClassPathBeanDefinitionScanner.java:345) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ClassPathBeanDefinitionScanner.doScan(ClassPathBeanDefinitionScanner.java:283) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ComponentScanAnnotationParser.parse(ComponentScanAnnotationParser.java:135) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ConfigurationClassParser.doProcessConfigurationClass(ConfigurationClassParser.java:287) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ConfigurationClassParser.processConfigurationClass(ConfigurationClassParser.java:245) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ConfigurationClassParser.parse(ConfigurationClassParser.java:198) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    at org.springframework.context.annotation.ConfigurationClassParser.parse(ConfigurationClassParser.java:167) ~[spring-context-4.3.10.RELEASE.jar:4.3.10.RELEASE]
    ... 13 common frames omitted


```
