---
title: Gradle 使用-添加 Jacoco 分析单元测试覆盖率工具
date: 2018-01-01 00:03:40
tags:
    - Java
    - Gradle
    - SpringBoot 
    - Junit
    - Test
    - Jacoco
categories: 
    - Java
    - Gradle
    - SpringBoot
    - Junit
    - Test
    - Jacoco
---
# Gradle 添加 Jacoco 分析单元测试覆盖率工具
> Jacoco  是一个免费的 Java 单元测试覆盖率分析工具，在 Gradle 中添加插件，在编译的同事进行单元测试覆盖率分析

## 配置

```
buildscript {
    repositories {
        mavenCentral()
        maven { url "https://plugins.gradle.org/m2/" }
    }
}

apply plugin: 'java'
apply plugin: 'jacoco'


group = 'cn.com.hellowood'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
    mavenCentral()
    maven { url "https://plugins.gradle.org/m2/" }
}

war {
    baseName = 'Security'
    version = ''
}

jacocoTestReport {
    reports {
        xml.enabled false
        html.enabled true
    }
}

check.dependsOn jacocoTestReport
```

## 生成结果
> 编译完成后会在 `${buildDir}/build/reports/jacoco/` 下会生成报告

![Jacoco测试结果](http://img.blog.csdn.net/20171207221854097?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


