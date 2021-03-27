---
title: IDEA中Spring Boot项目MyBaits提示Invalid bound statement (not found)错误
date: 2018-01-01 01:01:51
tags:
    - Java
    - SpringBoot
    - MyBatis
    - Exception 
categories: 
    - Java
    - SpringBoot
    - MyBatis
    - Exception
---
> 一个SpringBoot项目在STS中是正常的，没有任何问题，但是导入到IDEA中之后启动就提示`org.apache.ibatis.binding.BindingException: Invalid bound statement (not found)`错误

```
2017-05-01 20:29:30.089 ERROR 8580 --- [nio-8080-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): hellowood.lntu.oe.wmp.dao.FeedbackDetailMapper.insertSelective] with root cause

org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): ...........
```
> 该错误提示没有找到相对应的XML文件，找了很长时间发现在编译后的classes路径下并没有相应的XML文件，这是因为IDEA在编译的时候忽略掉了XML文件，一个解决方法是将所有的XML文件移动到Resource文件夹下，这样在编译的时候就会将XML文件一起

- 移动文件夹后修改配置文件中的MyBat的扫描路径

```
 mybatis.mapper-locations=classpath*:/mapper/**Mapper.xml
```
- 修改前的结构
![修改前的结构](http://img.blog.csdn.net/20170501203323011?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 修改后的结构
![修改后的结构](http://img.blog.csdn.net/20170501203402426?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
