---
title: >-
  MyBatis 查询错误：java.lang.IllegalArgumentException: invalid comparison:
  java.util.Date and java.lang.String
date: 2019-03-29 21:17:57
tags:
    - MyBatis 
    - Java
    - SpringBoot
categories: 
    - MyBatis 
    - Java
    - SpringBoot
---

# MyBatis 查询错误：java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String


项目中用 MyBatis Plus 替换了 MyBatis，原来的 MyBatis版本是 `3.2.8`, MyBatis Plus 的版本是 `3.1.0`，是基于 MyBatis `3.5.0`开发的，测试没啥问题，上线之后有一些功能不能使用，排查日志发现错误`Error querying database.  Cause: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String`

很快就定位到问题，是因为传入的 Date 在 XML 中使用了 `date != null and date != ''`这样的判断引起的，删除`date != ''`这个判断就好了

- 异常信息

```java
Caused by: org.apache.ibatis.exceptions.PersistenceException: 
### Error querying database.  Cause: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String
### Cause: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String
    at org.apache.ibatis.exceptions.ExceptionFactory.wrapException(ExceptionFactory.java:30)
    at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:150)
    at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:141)
    at org.apache.ibatis.session.defaults.DefaultSqlSession.selectOne(DefaultSqlSession.java:77)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:433)
    ... 98 more
```

### 问题复现

- ServiceImpl

```java
testDao.getDate(new Date());
```

- Dao

```java
Date test(@Param("date") Date date);
```

- XML

```xml
    <select id="test" resultType="java.util.Date">
        <if test="date != null and date !=''" >
            SELECT
            #{date} AS now
            FROM dual
        </if>
        <if test=" date== null">
            SELECT
            date_add(now(), INTERVAL 1 DAY_HOUR) AS now
            FROM dual
        </if>
    </select>
```

经过debug，发现 MyBatis 3.3之前和3.3之后的判断逻辑不一样

在3.3之前，判断逻辑是：非基本类型的包装类型，在 `compareWithConversion`方法里传入的 equals 这个参数为 true，发现比较对象的class不一样时，不抛出异常，返回值为1，即：

```java
if (!equals) {
    throw new IllegalArgumentException("invalid comparison: " + v1.getClass().getName() + " and " + v2.getClass().getName());
}

result = 1;
```

通过`compareWithConversion(object1, object2, true) == 0 || object1.equals(object2);` 得出的结果是false，即判断条件不成立；

在3.3之后，在 `compareWithConversion`方法中移除了 equals 这个参数，判断的逻辑改成了 `result = object1 != null && object2 != null && (object1.equals(object2) || compareWithConversion(object1, object2) == 0);`，`compareWithConversion`方法里当比较的对象 class 不同的时候，会抛出 `IllegalArgumentException`


以下是两个版本的比较部分的逻辑代码：

##### MyBatis 3.2.8 相关代码 

- org.apache.ibatis.ognl.OgnlOps#isEqual:76 

```java
public static boolean isEqual(Object object1, Object object2) {
        boolean result = false;
        if (object1 == object2) {
            result = true;
        } else if (object1 != null && object2 != null) {
            if (object1.getClass().isArray() && object2.getClass().isArray() && object2.getClass() == object1.getClass()) {
                result = Array.getLength(object1) == Array.getLength(object2);
                if (result) {
                    int i = 0;

                    for(int icount = Array.getLength(object1); result && i < icount; ++i) {
                        result = isEqual(Array.get(object1, i), Array.get(object2, i));
                    }
                }
            } else if (object1 != null && object2 != null) {
                result = compareWithConversion(object1, object2, true) == 0 || object1.equals(object2);
            }
        }

        return result;
    }
```


- org.apache.ibatis.ognl.OgnlOps#compareWithConversion:21

```java
            case 10:
                if (t1 == 10 && t2 == 10) {
                    if (v1 != null && v2 != null) {
                        if (v1.getClass().isAssignableFrom(v2.getClass()) || v2.getClass().isAssignableFrom(v1.getClass())) {
                            if (v1 instanceof Comparable) {
                                result = ((Comparable)v1).compareTo(v2);
                                break;
                            }

                            if (equals) {
                                result = v1.equals(v2) ? 0 : 1;
                                break;
                            }
                        }

                        if (!equals) {
                            throw new IllegalArgumentException("invalid comparison: " + v1.getClass().getName() + " and " + v2.getClass().getName());
                        }

                        result = 1;
                        break;
                    } else {
                        boolean var10000 = v1 != v2;
                    }
                }
```

##### MyBatis 3.5.0 相关代码 

- org.apache.ibatis.ognl.OgnlOps#isEqual:58

```java
    public static boolean isEqual(Object object1, Object object2) {
        boolean result = false;
        if (object1 == object2) {
            result = true;
        } else if (object1 != null && object1.getClass().isArray()) {
            if (object2 != null && object2.getClass().isArray() && object2.getClass() == object1.getClass()) {
                result = Array.getLength(object1) == Array.getLength(object2);
                if (result) {
                    int i = 0;

                    for(int icount = Array.getLength(object1); result && i < icount; ++i) {
                        result = isEqual(Array.get(object1, i), Array.get(object2, i));
                    }
                }
            }
        } else {
            result = object1 != null && object2 != null && (object1.equals(object2) || compareWithConversion(object1, object2) == 0);
        }

        return result;
    }
```

- org.apache.ibatis.ognl.OgnlOps#compareWithConversion:19

```java
    case 10:
    if (t1 == 10 && t2 == 10) {
        if (v1 instanceof Comparable && v1.getClass().isAssignableFrom(v2.getClass())) {
                result = ((Comparable)v1).compareTo(v2);
                break;
        }

       throw new IllegalArgumentException("invalid comparison: " + v1.getClass().getName() + " and " + v2.getClass().getName());
    }
```

### 总结

这个问题来自两方面：
1. 开发不规范，使用 Date 和 String 做比较，这个本身就是有问题的， 类似的还有用 String 比较集合或者对象；
2. MyBatis 3.3 之前的 bug 导致在比较不同类型的时候，本来应该抛出的异常没有被抛出，导致这个问题被隐藏

总结下来就是还是因为开发不规范导致的问题，因为本身判断 `!= null` 就不应该再判断 `!=''`，因为`""`本身是有值的，应该存到数据库中，除非是特殊要求；在这个项目中是因为前台提交的表单中没有值的字段被参数接收后变成了 `""`而不是 `null`，但是这个问题应该是通过在 Controller 相关类中修改 `WebDataBinder`解决的：

```java
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(String.class, new StringTrimmerEditor(true));
    }
```

#### 相关 ISSUE

- [java.util.RandomAccessSubList and java.lang.String](https://github.com/mybatis/mybatis-3/issues/445)
- [3.3.1 java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String](https://github.com/mybatis/mybatis-3/issues/1340) 

----------

#### 调用栈

```java
compareWithConversion:130, OgnlOps (org.apache.ibatis.ognl)
isEqual:186, OgnlOps (org.apache.ibatis.ognl)
equal:578, OgnlOps (org.apache.ibatis.ognl)
getValueBody:51, ASTNotEq (org.apache.ibatis.ognl)
evaluateGetValueBody:170, SimpleNode (org.apache.ibatis.ognl)
getValue:210, SimpleNode (org.apache.ibatis.ognl)
getValueBody:56, ASTAnd (org.apache.ibatis.ognl)
evaluateGetValueBody:170, SimpleNode (org.apache.ibatis.ognl)
getValue:210, SimpleNode (org.apache.ibatis.ognl)
getValue:333, Ognl (org.apache.ibatis.ognl)
getValue:310, Ognl (org.apache.ibatis.ognl)
getValue:45, OgnlCache (org.apache.ibatis.scripting.xmltags)
evaluateBoolean:32, ExpressionEvaluator (org.apache.ibatis.scripting.xmltags)
apply:33, IfSqlNode (org.apache.ibatis.scripting.xmltags)
apply:32, MixedSqlNode (org.apache.ibatis.scripting.xmltags)
getBoundSql:40, DynamicSqlSource (org.apache.ibatis.scripting.xmltags)
getBoundSql:278, MappedStatement (org.apache.ibatis.mapping)
query:75, CachingExecutor (org.apache.ibatis.executor)
invoke:-1, GeneratedMethodAccessor338 (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
proceed:49, Invocation (org.apache.ibatis.plugin)
invoke:60, Plugin (org.apache.ibatis.plugin)
query:-1, $Proxy470 (com.sun.proxy)
invoke:-1, GeneratedMethodAccessor338 (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
proceed:49, Invocation (org.apache.ibatis.plugin)
invoke:60, Plugin (org.apache.ibatis.plugin)
query:-1, $Proxy470 (com.sun.proxy)
selectList:108, DefaultSqlSession (org.apache.ibatis.session.defaults)
selectList:102, DefaultSqlSession (org.apache.ibatis.session.defaults)
selectOne:66, DefaultSqlSession (org.apache.ibatis.session.defaults)
invoke:-1, GeneratedMethodAccessor340 (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
invoke:358, SqlSessionTemplate$SqlSessionInterceptor (org.mybatis.spring)
selectOne:-1, $Proxy280 (com.sun.proxy)
selectOne:163, SqlSessionTemplate (org.mybatis.spring)
execute:68, MapperMethod (org.apache.ibatis.binding)
invoke:52, MapperProxy (org.apache.ibatis.binding)
test:-1, $Proxy288 (com.sun.proxy)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
invokeJoinpointUsingReflection:333, AopUtils (org.springframework.aop.support)
invokeJoinpoint:190, ReflectiveMethodInvocation (org.springframework.aop.framework)
proceed:157, ReflectiveMethodInvocation (org.springframework.aop.framework)
proceed:85, MethodInvocationProceedingJoinPoint (org.springframework.aop.aspectj)
invoke:-1, GeneratedMethodAccessor339 (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
invokeAdviceMethodWithGivenArgs:629, AbstractAspectJAdvice (org.springframework.aop.aspectj)
invokeAdviceMethod:618, AbstractAspectJAdvice (org.springframework.aop.aspectj)
invoke:70, AspectJAroundAdvice (org.springframework.aop.aspectj)
proceed:179, ReflectiveMethodInvocation (org.springframework.aop.framework)
invoke:92, ExposeInvocationInterceptor (org.springframework.aop.interceptor)
proceed:179, ReflectiveMethodInvocation (org.springframework.aop.framework)
invoke:213, JdkDynamicAopProxy (org.springframework.aop.framework)
test:-1, $Proxy289 (com.sun.proxy)
test:453, SystemServiceImpl (io.helloworlde.github.service)
invoke:-1, SystemServiceImpl$$FastClassBySpringCGLIB$$ccedf5b2 (io.helloworlde.github.service)
invoke:204, MethodProxy (org.springframework.cglib.proxy)
intercept:652, CglibAopProxy$DynamicAdvisedInterceptor (org.springframework.aop.framework)
test:-1, SystemServiceImpl$$EnhancerBySpringCGLIB$$c36d45a6 (io.helloworlde.github.service)
test:155, PersonalCenterController (io.helloworlde.github.controller)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
doInvoke:205, InvocableHandlerMethod (org.springframework.web.method.support)
invokeForRequest:133, InvocableHandlerMethod (org.springframework.web.method.support)
invokeAndHandle:116, ServletInvocableHandlerMethod (org.springframework.web.servlet.mvc.method.annotation)
invokeHandlerMethod:827, RequestMappingHandlerAdapter (org.springframework.web.servlet.mvc.method.annotation)
handleInternal:738, RequestMappingHandlerAdapter (org.springframework.web.servlet.mvc.method.annotation)
handle:85, AbstractHandlerMethodAdapter (org.springframework.web.servlet.mvc.method)
doDispatch:963, DispatcherServlet (org.springframework.web.servlet)
doService:897, DispatcherServlet (org.springframework.web.servlet)
processRequest:970, FrameworkServlet (org.springframework.web.servlet)
doGet:861, FrameworkServlet (org.springframework.web.servlet)
service:634, HttpServlet (javax.servlet.http)
service:846, FrameworkServlet (org.springframework.web.servlet)
service:741, HttpServlet (javax.servlet.http)
internalDoFilter:231, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
doFilter:53, WsFilter (org.apache.tomcat.websocket.server)
internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
doFilter:123, WebStatFilter (com.alibaba.druid.support.http)
internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
doFilterInternal:105, HttpPutFormContentFilter (org.springframework.web.filter)
doFilter:107, OncePerRequestFilter (org.springframework.web.filter)
internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
executeChain:449, AbstractShiroFilter (org.apache.shiro.web.servlet)
call:365, AbstractShiroFilter$1 (org.apache.shiro.web.servlet)
doCall:90, SubjectCallable (org.apache.shiro.subject.support)
call:83, SubjectCallable (org.apache.shiro.subject.support)
execute:383, DelegatingSubject (org.apache.shiro.subject.support)
doFilterInternal:362, AbstractShiroFilter (org.apache.shiro.web.servlet)
doFilter:125, OncePerRequestFilter (org.apache.shiro.web.servlet)
invokeDelegate:346, DelegatingFilterProxy (org.springframework.web.filter)
doFilter:262, DelegatingFilterProxy (org.springframework.web.filter)
internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
doFilterInternal:164, SessionRepositoryFilter (org.springframework.session.web.http)
doFilter:80, OncePerRequestFilter (org.springframework.session.web.http)
invokeDelegate:346, DelegatingFilterProxy (org.springframework.web.filter)
doFilter:262, DelegatingFilterProxy (org.springframework.web.filter)
internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
doFilterInternal:197, CharacterEncodingFilter (org.springframework.web.filter)
doFilter:107, OncePerRequestFilter (org.springframework.web.filter)
internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
invoke:199, StandardWrapperValve (org.apache.catalina.core)
invoke:96, StandardContextValve (org.apache.catalina.core)
invoke:475, AuthenticatorBase (org.apache.catalina.authenticator)
invoke:140, StandardHostValve (org.apache.catalina.core)
invoke:80, ErrorReportValve (org.apache.catalina.valves)
invoke:625, AbstractAccessLogValve (org.apache.catalina.valves)
invoke:87, StandardEngineValve (org.apache.catalina.core)
service:342, CoyoteAdapter (org.apache.catalina.connector)
service:498, Http11Processor (org.apache.coyote.http11)
process:66, AbstractProcessorLight (org.apache.coyote)
process:796, AbstractProtocol$ConnectionHandler (org.apache.coyote)
doRun:1372, NioEndpoint$SocketProcessor (org.apache.tomcat.util.net)
run:49, SocketProcessorBase (org.apache.tomcat.util.net)
runWorker:1149, ThreadPoolExecutor (java.util.concurrent)
run:624, ThreadPoolExecutor$Worker (java.util.concurrent)
run:61, TaskThread$WrappingRunnable (org.apache.tomcat.util.threads)
run:748, Thread (java.lang)
```

#### 异常堆栈 

```java
### Error querying database.  Cause: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String
### Cause: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String
    at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:77)
    at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:446)
    at com.sun.proxy.$Proxy51.selectOne(Unknown Source)
    at org.mybatis.spring.SqlSessionTemplate.selectOne(SqlSessionTemplate.java:166)
    at com.baomidou.mybatisplus.core.override.MybatisMapperMethod.execute(MybatisMapperMethod.java:99)
    at com.baomidou.mybatisplus.core.override.MybatisMapperProxy.invoke(MybatisMapperProxy.java:61)
    at com.sun.proxy.$Proxy61.test(Unknown Source)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:333)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.invokeJoinpoint(ReflectiveMethodInvocation.java:190)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157)
    at org.springframework.aop.aspectj.MethodInvocationProceedingJoinPoint.proceed(MethodInvocationProceedingJoinPoint.java:85)
    at sun.reflect.GeneratedMethodAccessor193.invoke(Unknown Source)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:629)
    at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:618)
    at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:70)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
    at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
    at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:213)
    at com.sun.proxy.$Proxy62.test(Unknown Source)
    at io.helloworlde.github.service.SystemServiceImpl.test(SystemServiceImpl.java:453)
    at io.helloworlde.github.service.SystemServiceImpl$$FastClassBySpringCGLIB$$ccedf5b2.invoke(<generated>)
    at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
    at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:652)
    at io.helloworlde.github.service.SystemServiceImpl$$EnhancerBySpringCGLIB$$f63e5835.getSystemConfiguration(<generated>)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:205)
    at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:133)
    at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:116)
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:827)
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:738)
    at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:963)
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:897)
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
    at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:861)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at com.alibaba.druid.support.http.WebStatFilter.doFilter(WebStatFilter.java:123)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:105)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.shiro.web.servlet.AbstractShiroFilter.executeChain(AbstractShiroFilter.java:449)
    at org.apache.shiro.web.servlet.AbstractShiroFilter$1.call(AbstractShiroFilter.java:365)
    at org.apache.shiro.subject.support.SubjectCallable.doCall(SubjectCallable.java:90)
    at org.apache.shiro.subject.support.SubjectCallable.call(SubjectCallable.java:83)
    at org.apache.shiro.subject.support.DelegatingSubject.execute(DelegatingSubject.java:383)
    at org.apache.shiro.web.servlet.AbstractShiroFilter.doFilterInternal(AbstractShiroFilter.java:362)
    at org.apache.shiro.web.servlet.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:125)
    at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:346)
    at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:262)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.session.web.http.SessionRepositoryFilter.doFilterInternal(SessionRepositoryFilter.java:164)
    at org.springframework.session.web.http.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:80)
    at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:346)
    at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:262)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:197)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199)
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:475)
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140)
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:80)
    at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:625)
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:342)
    at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:498)
    at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
    at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:796)
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1372)
    at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
    at java.lang.Thread.run(Thread.java:748)
Caused by: org.apache.ibatis.exceptions.PersistenceException: 
### Error querying database.  Cause: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String
### Cause: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String
    at org.apache.ibatis.exceptions.ExceptionFactory.wrapException(ExceptionFactory.java:30)
    at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:150)
    at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:141)
    at org.apache.ibatis.session.defaults.DefaultSqlSession.selectOne(DefaultSqlSession.java:77)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:433)
    ... 98 more
Caused by: java.lang.IllegalArgumentException: invalid comparison: java.util.Date and java.lang.String
    at org.apache.ibatis.ognl.OgnlOps.compareWithConversion(OgnlOps.java:93)
    at org.apache.ibatis.ognl.OgnlOps.isEqual(OgnlOps.java:143)
    at org.apache.ibatis.ognl.OgnlOps.equal(OgnlOps.java:802)
    at org.apache.ibatis.ognl.ASTNotEq.getValueBody(ASTNotEq.java:53)
    at org.apache.ibatis.ognl.SimpleNode.evaluateGetValueBody(SimpleNode.java:212)
    at org.apache.ibatis.ognl.SimpleNode.getValue(SimpleNode.java:258)
    at org.apache.ibatis.ognl.ASTAnd.getValueBody(ASTAnd.java:61)
    at org.apache.ibatis.ognl.SimpleNode.evaluateGetValueBody(SimpleNode.java:212)
    at org.apache.ibatis.ognl.SimpleNode.getValue(SimpleNode.java:258)
    at org.apache.ibatis.ognl.Ognl.getValue(Ognl.java:493)
    at org.apache.ibatis.ognl.Ognl.getValue(Ognl.java:457)
    at org.apache.ibatis.scripting.xmltags.OgnlCache.getValue(OgnlCache.java:46)
    at org.apache.ibatis.scripting.xmltags.ExpressionEvaluator.evaluateBoolean(ExpressionEvaluator.java:32)
    at org.apache.ibatis.scripting.xmltags.IfSqlNode.apply(IfSqlNode.java:34)
    at org.apache.ibatis.scripting.xmltags.MixedSqlNode.apply(MixedSqlNode.java:33)
    at org.apache.ibatis.scripting.xmltags.DynamicSqlSource.getBoundSql(DynamicSqlSource.java:41)
    at org.apache.ibatis.mapping.MappedStatement.getBoundSql(MappedStatement.java:293)
    at org.apache.ibatis.plugin.Plugin.invoke(Plugin.java:61)
    at com.sun.proxy.$Proxy248.query(Unknown Source)
    at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:148)
    ... 105 more
```