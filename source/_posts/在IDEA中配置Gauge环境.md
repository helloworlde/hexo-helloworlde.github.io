---
title: 在IDEA中配置Gauge环境
date: 2018-01-01 11:28:05
tags:
    - Java
    - Gauge
    - Test
categories: 
    - Java
    - Gauge
    - Test
---
> ### [Gauge](http://getgauge.io)是一个自动化测试工具，主要是通过.spec 文件指定执行的步骤，然后由Java代码去测试

- 首先，[下载](http://getgauge.io/get-started/index.html)安装Gauge
- 安装后通过`cmd`运行`guage -v` 来确认Gauge安装成功
![这里写图片描述](http://img.blog.csdn.net/20170119132307236?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- 安装成功后安装Gauge的插件
 - `gauge --install-all`
 - 或者可以用 `gauge --install java`和`gauge --insatll html-report`·

 ![这里写图片描述](http://img.blog.csdn.net/20170119132349174?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
-  在IDEA中下载Gauge插件安装
![这里写图片描述](http://img.blog.csdn.net/20170119132409063?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- 安装完成后重启IDEA即完成Gauge的环境配置


----------
注意：

-  如果修改了安装目录需要配置`GAUGE_ROOT`，否则IDEA会报错
- JDK的所有`PATH`，`JAVA_HOME`，`CLASSPATH`环境变量都需要配置好
- 需要确认`PATH`中配置了Gauge的`bin`目录