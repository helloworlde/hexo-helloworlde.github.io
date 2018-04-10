---
title: Java 将单元测试覆盖率上传到 Codecov
date: 2018-01-01 00:00:19
tags:
    - Java
    - Gradle
    - SpringBoot 
    - Junit
    - Test
    - Codecov
categories: 
    - Java
    - Gradle
    - SpringBoot
    - Junit
    - Test
    - Codecov
---

# 将单元测试覆盖率上传到 Codecov

> 通过使用 [Jacoco](http://www.eclemma.org/jacoco/) 生成单元测试覆盖率报告，并将该报告上传到 [Codecov](https://codecov.io)

## 配置 Jacoco

- 配置 build.gradle 文件(以 SpringBoot 应用为例)

```groovy
buildscript {
    ext {
        springBootVersion = '1.5.8.RELEASE'
    }
    repositories {
        mavenCentral()
        maven { url "https://plugins.gradle.org/m2/" }
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("gradle.plugin.com.gorylenko.gradle-git-properties:gradle-git-properties:1.4.17")
    }
}

apply plugin: 'java'
apply plugin: 'eclipse-wtp'
apply plugin: 'org.springframework.boot'
apply plugin: 'war'

// 使用 Jacoco 插件
apply plugin: 'jacoco'


group = 'cn.com.hellowood'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

war {
    baseName = 'Security'
    version = ''
}

dependencies {
    compile('org.springframework.boot:spring-boot-starter-security')
    compile('org.springframework.boot:spring-boot-starter-web')
    compile("org.springframework.boot:spring-boot-devtools")
    compile('org.mybatis.spring.boot:mybatis-spring-boot-starter:1.3.0')
    compile('org.springframework.boot:spring-boot-starter-thymeleaf')
    compile('org.webjars:jquery:3.2.1')
    compile('org.webjars:bootstrap:4.0.0-beta.2')
    compile('org.webjars:font-awesome:4.7.0')
    compile('org.webjars:bootstrap-glyphicons:bdd2cbfba0')
    runtime('mysql:mysql-connector-java')
    runtime('org.springframework.boot:spring-boot-starter-tomcat')
    testCompile('org.springframework.boot:spring-boot-starter-test')
    testCompile('org.springframework.security:spring-security-test')
}

// 添加 task 用于生成 Jacoco 测试结果
task codeCoverageReport(type: JacocoReport) {
    
    // 需指定生成的类文件位置和源文件位置
    classDirectories = files('build/classes')
    sourceDirectories = files('src/main/java')

    executionData fileTree(project.rootDir.absolutePath).include("**/build/jacoco/*.exec")

    subprojects.each {
        sourceSets it.sourceSets.main
    }

    // 生成的报告类型包括 xml/html/csv
    reports {
        xml.enabled true
        xml.destination "${buildDir}/reports/jacoco/report.xml"
        html.enabled false
        csv.enabled false
    }
}

codeCoverageReport.dependsOn {
    subprojects*.test
}

check.dependsOn codeCoverageReport

```

## 生成测试报告

```groovy
    gradle check
    gradle codeCoverageReport
```
> 此时在 `build/reports/jacoco` 下生成 Jacoco 的测试报告

## 上传测试结果

- 通过命令直接上传(TOKEN 在 Codecov 项目中可以找到)

```
    bash <(curl -s https://codecov.io/bash) -t YOUR_PROJECT_TOKEN
```

- 通过 Travis CI 上传(在 Travis 配置文件中添加以下内容)

```
    script:
      - ./gradlew check
      - ./gradlew codeCoverageReport
    after_success:
      - codecov
      - bash <(curl -s https://codecov.io/bash) -t YOUR_PROJECT_TOKEN
```

- 通过 Circle CI 上传(在 Circle 配置文件中添加以下内容)

```
    jobs:
      build:
        steps:
          - run: gradle check
          - run: gradle codeCoverageReport
          - run: bash <(curl -s https://codecov.io/bash) -t 92b3ad6b-92f7-49bb-94e7-ea233b860e47
```


--------------

> 需要注意的是只有 xml 格式的测试报告才会被上传

![分析结果](http://img.blog.csdn.net/20171208203442891?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)