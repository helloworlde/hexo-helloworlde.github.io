---
title: 使用Gradle编译执行Gague项目
date: 2018-01-01 11:32:34
tags:
    - Java
    - Gradle
    - Gauge
    - Test
categories: 
    - Java
    - Gradle
    - Gauge
    - Test
---
使用Gradle编译运行Gauge项目可以很大程度解决依赖的问题，并且可以根据需要创建多个不同的Task来在不同的环境运行或执行不同的操作

------------------

## 创建Gauge项目
- 首先在IDEA中创建一个Gauge项目
![这里写图片描述](http://img.blog.csdn.net/20170216213149019?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- 打开命令行，执行`gradle init` 初始化Gradle项目
![这里写图片描述](http://img.blog.csdn.net/20170216213209779?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- 修改build.gradle文件，添加Gauge的依赖

```
apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'gauge'

group = "Gradle-Gauge"
version = "1.0.0"


sourceCompatibility = 1.7
targetCompatibility = 1.7

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.thoughtworks.gauge.gradle:gauge-gradle-plugin:+'
    }
}

repositories {
    mavenCentral()
}

dependencies {
    //添加selenium是为了执行网页测试
    compile(
            'com.thoughtworks.gauge:gauge-java:0.5.1',
            'junit:junit:4.12',
            'org.seleniumhq.selenium:selenium-chrome-driver:3.0.1',
            'org.seleniumhq.selenium:selenium-support:3.0.1'            
    )
}

//执行`gradle gague`时是在执行该Task
gauge {
    specsDir = 'specs'
}
```

- 执行`gradle build`来编译项目，并下载依赖
- 执行`gradle gauge`来运行Gauge项目，执行测试