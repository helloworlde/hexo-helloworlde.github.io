---
title: Spring MVC简单实现国际化/多语言
date: 2018-01-01 11:48:46
tags:
    - Java
    - SrpingMVC 
    - i18n
categories: 
    - Java
    - SrpingMVC 
    - i18n
---
> SpringMVC 可以通过Spring框架来实现多语言

## 1. 创建SpringMVC项目

- 配置web.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">

    <!--Spring 配置文件-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>

    <!--监听器-->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!--配置转发器-->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>*.action</url-pattern>
    </servlet-mapping>

    <filter>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>
```

- 配置Spring文件(dispatcher-servlet.xml)

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <!-- 默认使用基于注释的适配器和映射器 -->
    <mvc:annotation-driven/>
    <!-- 只把动态信息当做controller处理，忽略静态信息 -->
    <mvc:default-servlet-handler/>
    <!-- 自动扫描包中的Controlller -->
    <context:component-scan base-package="controller"/>

    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
        <property name="prefix" value="/WEB-INF/jsp/"/><!-- 前缀 -->
        <property name="suffix" value=".jsp"/><!-- 后缀，自动拼接 -->
    </bean>
</beans>
```
    
## 2. 添加多语言的配置文件

- 添加language_en_US.properties到src目录下

```
language.cn = \u4e2d\u6587
language.en = English
internationalisation = \u0020Internationalisation
welcome = This is the English environment
introduce= This is I18N Demo 
```

- 添加language_zh_CN.properties到src目录下

```
language.cn = \u4e2d\u6587
language.en = English
internationalisation = \u56fd\u9645\u5316
welcome = \u8fd9\u662f\u4e2d\u6587\u73af\u5883
introduce= \u8fd9\u662f\u56fd\u9645\u5316\u7684\u4e8b\u4f8b
```

## 3. 加入i18n 过滤器到配置文件中
- 将配置添加到dispatcher-servelet中
```
    <!-- 国际化资源文件 -->
    <bean id="messageSource" class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
        <!-- 表示多语言配置文件在根路径下，以language开头的文件-->
        <property name="basename" value="classpath:language"/>
        <property name="useCodeAsDefaultMessage" value="true"/>
    </bean>

    <mvc:interceptors>
        <bean id="localeChangeInterceptor" class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
            <property name="paramName" value="lang"/>
        </bean>
    </mvc:interceptors>
```
## 4. 在页面中使用多语言

- 在Controller中添加路径

```
@Controller
public class HelloController {
    @RequestMapping("/hello.action")
    public String index() {
        return "hello";
    }
}
```

- 在JSP页面中使用
> 通过`<spring:message code="welcome"/>`将配置文件中的内容读取
```
<%@ page language="java" contentType="text/html; charset=UTF-8"%>
<%@taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<html>
<head>
    <title>SpringMVC<spring:message code="internationalisation"/></title>
</head>
<body>
    Language:
    <a href="?lang=zh_CN"><spring:message code="language.cn"/></a> &nbsp;&nbsp;&nbsp;
    <a href="?lang=en_US"><spring:message code="language.en"/></a>
    <h1>
        <spring:message code="welcome"/>
    </h1>
    当前语言: ${pageContext.response.locale }
</body>
</html>
```

- 访问该项目的`/hello.action`，通过链接可以切换语言


----------

- 中文
![cn](http://img.blog.csdn.net/20170427170202803?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- 英文
![en](http://img.blog.csdn.net/20170427170214553?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- 项目结构
![项目结构](http://img.blog.csdn.net/20170427170229926?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


----------
##[效果演示](http://project.hellowood.com.cn:8080/i18n/)
##[项目下载](http://download.csdn.net/detail/u013360850/9827744)
