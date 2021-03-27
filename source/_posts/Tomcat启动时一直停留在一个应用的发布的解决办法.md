---
title: Tomcat启动时一直停留在一个应用的发布的解决办法
date: 2018-01-01 01:05:02
tags:
    - Tomcat
categories: 
    - Tomcat    

---
> Tomcat在启动时一直停留在某一个应用无法启动或者需要很长时间才能启动，提示`Deploying web application directory [/home/dev/tomcat/apache-tomcat-9.0.0.M26/webapps/ROOT`，可以通过如下配置来加速启动

## 配置
###修改**`${JAVA_HOME}/jre/lib/security/java.security`**文件
###修改**`securerandom.source=file:/dev/random`**为**`securerandom.source=file:/dev/./urandom`**即可

## 解释
> 这是因为Tomcat 7以上的版本在启动的时候会使用 `org.apache.catalina.util.SessionIdGeneratorBase.createSecureRandom`类产生安全随机类`SecureRandom`的实例作为会话ID
> SHA1PRNG算法是基于SHA-1算法实现且保密性较强的伪随机数生成器。

> 在SHA1PRNG中，有一个种子产生器，它根据配置执行各种操作。

> Linux中的随机数可以从两个特殊的文件中产生，一个是/dev/urandom.另外一个是/dev/random。他们产生随机数的原理是利用当前系统的熵池来计算出固定一定数量的随机比特，然后将这些比特作为字节流返回。熵池就是当前系统的环境噪音，熵指的是一个系统的混乱程度，系统噪音可以通过很多参数来评估，如内存的使用，文件的使用量，不同类型的进程数量等等。如果当前环境噪音变化的不是很剧烈或者当前环境噪音很小，比如刚开机的时候，而当前需要大量的随机比特，这时产生的随机数的随机效果就不是很好了。

> 这就是为什么会有/dev/urandom和/dev/random这两种不同的文件，后者在不能产生新的随机数时会阻塞程序，而前者不会（ublock），当然产生的随机数效果就不太好了，这对加密解密这样的应用来说就不是一种很好的选择。/dev/random会阻塞当前的程序，直到根据熵池产生新的随机字节之后才返回，所以使用/dev/random比使用/dev/urandom产生大量随机数的速度要慢。

---------------
具体参考
[https://wiki.apache.org/tomcat/HowTo/FasterStartUp](https://wiki.apache.org/tomcat/HowTo/FasterStartUp)
[https://my.oschina.net/wangnian/blog/687914](https://my.oschina.net/wangnian/blog/687914)