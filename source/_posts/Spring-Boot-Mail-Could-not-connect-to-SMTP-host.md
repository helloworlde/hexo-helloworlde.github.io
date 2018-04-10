---
title: Spring Boot Mail Could not connect to SMTP host
date: 2017-12-31 23:57:39
tags:
    - Java
    - SpringBoot 
    - Mail
    - Exception
categories: 
    - Java
    - SpringBoot
    - Mail
    - Exception
---
> 如果使用了 SSL 连接请添加配置：`spring.mail.properties.mail.smtp.ssl.enable=true`
> 可以参考 [https://stackoverflow.com/questions/31721298/spring-boot-1-2-5-release-sending-e-mail-via-gmail-smtp](https://stackoverflow.com/questions/31721298/spring-boot-1-2-5-release-sending-e-mail-via-gmail-smtp)

-------------

> 在使用 Spring Boot 发送邮件时遇到了无法连接服务器的问题，使用的是阿里云的邮件服务，配置如下：

- application.properties

```properties
spring.mail.host=smtpdm.aliyun.com
spring.mail.port=465
spring.mail.username=username
spring.mail.password=password
spring.mail.properties.smtp.auth=true
spring.mail.properties.smtp.starttls.enable=true
```

- MailUtil.java：


```java
   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.mail.MailException;
   import org.springframework.mail.SimpleMailMessage;
   import org.springframework.mail.javamail.JavaMailSender;
   import org.springframework.stereotype.Component;
   import org.thymeleaf.TemplateEngine;
   
   @Component
   public class MailUtil {
   
       private final Logger logger = LoggerFactory.getLogger(getClass());
   
       @Autowired
       JavaMailSender mailSender;
   
       @Autowired
       TemplateEngine templateEngine;
   
       public void sendSimpleEmail(String deliver, String[] receiver, String[] carbonCopy, String subject, String content) throws ServiceException {
   
           long startTimestamp = System.currentTimeMillis();
           logger.info("Start send mail ... ");
   
           try {
               SimpleMailMessage message = new SimpleMailMessage();
               message.setFrom(deliver);
               message.setTo(receiver);
               message.setCc(carbonCopy);
               message.setSubject(subject);
               message.setText(content);
               mailSender.send(message);
               logger.info("Send mail success, cost {} million seconds", System.currentTimeMillis() - startTimestamp);
           } catch (MailException e) {
               logger.error("Send mail failed, error message is : {} \n", e.getMessage());
               e.printStackTrace();
               throw new ServiceException(e.getMessage());
           }
       }
   
   }
```

> 但是邮件发送一直失败，异常信息如下：

```java

org.springframework.mail.MailSendException: Mail server connection failed; nested exception is javax.mail.MessagingException: Could not connect to SMTP host: smtpdm.aliyun.com, port: 465, response: -1. Failed messages: javax.mail.MessagingException: Could not connect to SMTP host: smtpdm.aliyun.com, port: 465, response: -1; message exception details (1) are:
Failed message 1:
javax.mail.MessagingException: Could not connect to SMTP host: smtpdm.aliyun.com, port: 465, response: -1
    at com.sun.mail.smtp.SMTPTransport.openServer(SMTPTransport.java:2106)
    at com.sun.mail.smtp.SMTPTransport.protocolConnect(SMTPTransport.java:712)
    at javax.mail.Service.connect(Service.java:366)
    at org.springframework.mail.javamail.JavaMailSenderImpl.connectTransport(JavaMailSenderImpl.java:501)
    at org.springframework.mail.javamail.JavaMailSenderImpl.doSend(JavaMailSenderImpl.java:421)
    at org.springframework.mail.javamail.JavaMailSenderImpl.send(JavaMailSenderImpl.java:307)
    at org.springframework.mail.javamail.JavaMailSenderImpl.send(JavaMailSenderImpl.java:296)
    at cn.com.hellowood.mail.util.MailUtil.sendSimpleEmail(MailUtil.java:76)
    at cn.com.hellowood.mail.MailUtilTests.sendSimpleEmail(MailUtilTests.java:48)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
    at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
    at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
    at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
    at org.springframework.test.context.junit4.statements.RunBeforeTestMethodCallbacks.evaluate(RunBeforeTestMethodCallbacks.java:75)
    at org.springframework.test.context.junit4.statements.RunAfterTestMethodCallbacks.evaluate(RunAfterTestMethodCallbacks.java:86)
    at org.springframework.test.context.junit4.statements.SpringRepeat.evaluate(SpringRepeat.java:84)
    at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:325)
    at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:252)
    at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:94)
    at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
    at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
    at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
    at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
    at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
    at org.springframework.test.context.junit4.statements.RunBeforeTestClassCallbacks.evaluate(RunBeforeTestClassCallbacks.java:61)
    at org.springframework.test.context.junit4.statements.RunAfterTestClassCallbacks.evaluate(RunAfterTestClassCallbacks.java:70)
    at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
    at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.run(SpringJUnit4ClassRunner.java:191)
    at org.junit.runners.Suite.runChild(Suite.java:128)
    at org.junit.runners.Suite.runChild(Suite.java:27)
    at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
    at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
    at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
    at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
    at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
    at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
    at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
    at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:68)
    at com.intellij.rt.execution.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:47)
    at com.intellij.rt.execution.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:242)
    at com.intellij.rt.execution.junit.JUnitStarter.main(JUnitStarter.java:70)

```

> 将端口改为80或25时可以正常发送，没有任何问题，所以问题在于使用465端口时 SSL 的配置出错了，添加了以下配置后问题解决：

```
spring.mail.properties.mail.smtp.ssl.enable=true
```