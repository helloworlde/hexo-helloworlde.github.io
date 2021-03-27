---
title: Gauge中Gradle自定义Task失败的解决方法
date: 2018-01-01 11:26:41
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
##Gauge中加入了Gradle之后根据官方文档自定义task并不能执行

```groovy
task gaugeTest(type: GaugeTask) {
    doFirst {
        gauge {
            specsDir = 'specs'
            inParallel = true
            nodes = 4
            env = 'test'
            additionalFlags = '--verbose'
        }
    }
}
```
- 错误信息

```console
    * What went wrong:
    A problem occurred evaluating root project 'Gauge'.
    > Could not find property 'GaugeTask' on root project 'Gauge'.

```
> 这是因为Gradle并不能识别GaugeTask，需要写GaugeTask的限定类名：

## 解决方法
 - 在build.gradle中添加

``` groovy
    import com.thoughtworks.gauge.gradle.GaugeTask
```


- 或者写成：
```groovy
task gaugeTest(type: com.thoughtworks.gauge.gradle.GaugeTask) {
    doFirst {
        gauge {
            specsDir = 'specs'
            inParallel = true
            nodes = 4
            env = 'test'
            additionalFlags = '--verbose'
        }
    }
}
```
> 这样就可以识别执行了
