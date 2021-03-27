---
title: SpringBoot中Junit测试注入Bean失败的解决方法
date: 2018-01-01 11:46:57
tags:
    - Java
    - SpringBoot 
    - Junit
categories: 
    - Java
    - SpringBoot
    - Junit
---
> 在SpringBoot中使用Junit做测试的时候测试DAO层的接口，但是一直提示注入Bean失败，报以下错误：

```
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'hellowood.TestFeedbackMapper': Unsatisfied dependency expressed through field 'feedbackDetailMapper'; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'hellowood.lntu.oe.wmp.dao.FeedbackDetailMapper' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true)}

```
> 在查询了其他项目的Junit后发现Junit的注解是这样的

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = Application.class)
@WebAppConfiguration
```
> 而新建的项目中是这样的

```
@RunWith(SpringRunner.class)
@SpringBootTest
@WebAppConfiguration
```
> 直接修改注解后发现不能引入`SpringApplicationConfiguration`，而所有的依赖只是版本不一样，查阅了Spring官方文档后发现新版中用`SpringBootTest`代替了`SpringApplicationConfiguration`，所以将注解改为以下形式就可以正常注入Bean了

```
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
@WebAppConfiguration
```