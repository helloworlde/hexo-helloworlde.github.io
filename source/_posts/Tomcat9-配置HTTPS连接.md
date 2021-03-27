---
title: Tomcat9 配置HTTPS连接
date: 2018-01-01 12:01:18
tags:
    - Tomcat
    - Https 
categories: 
    - Tomcat
    - Https
---
> Tomcat中配置HTTPS连接可以分为两步：
>       1. 生成证书
        2. 配置Tomcat

> 准备工作
> 
  - JDK
  - Tomcat

----------
# 1. 生成证书
> 证书可以使用Java来生成
 
 - 直接使用命令生成证书
 

```
keytool -genkeypair -alias "tomcat" -keyalg "RSA" -keystore "d:\DevConfig\tomcat.keystore"  
```

![生成Keystore](http://img.blog.csdn.net/20170424210011905?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 这样就会在`F:\`下生成一个`tomcat.keystore` 文件
> 密码在配置tomcat的时候会用到

# 2. 配置Tomcat
-  修改`TOMCAT_HOME\conf\server.xml`文件，将以下内容粘贴到Tomcat配置文件中

```
<Connector
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           port="8443" maxThreads="200"
           scheme="https" secure="true" SSLEnabled="true"
           keystoreFile="F:\tomcat.keystore" keystorePass="tomcat"
           clientAuth="false" sslProtocol="TLS"/>
```

- 保存后启动Tomcat，访问`https://localhost:8433`即可

 
![Tomcat Https](http://img.blog.csdn.net/20170424211734554?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 因为CA证书是自己生成的，不被浏览器认可，所以会被当做不安全网站，但不影响使用


----------
> 需要注意在配置文件有三种不同的实现方式 
   

```
    - JSSE （作为Java运行部分）
    - JSSE  （使用OpenSSL）
    - APR （使用OpenSSL）
```

> 这三种方式对应的配置文件并不一样，需要特别注意


>  另外`protocol`如果写成`HTTP/1.1`也会报错，应当使用以下三个中的一个，当使用APR的时候需要从[下载`tcnative-1.dll`](http://tomcat.apache.org/tomcat-7.0-doc/apr.html)放到Tomcat的bin目录下，否则会报错

```
org.apache.coyote.http11.Http11NioProtocol
org.apache.coyote.http11.Http11Nio2Protocol
org.apache.coyote.http11.Http11AprProtocol
```

    
 > 这里的配置是第一种方式，也是最简单的方式


----------
详细配置请看官方文档 [http://tomcat.apache.org/tomcat-9.0-doc/ssl-howto.html](http://tomcat.apache.org/tomcat-9.0-doc/ssl-howto.html)
