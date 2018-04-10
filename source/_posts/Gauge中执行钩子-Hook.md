---
title: Gauge中执行钩子(Hook)
date: 2018-01-01 11:31:40
tags:
    - Java
    - Gauge
    - Test
categories: 
    - Java
    - Gauge
    - Test
---
本文所有内容均参考自[Gauge官方文档](http://getgauge.io/documentation/user/current/language_features/execution_hooks.html)


----------
> 钩子可以理解为`Java`中的`AOP(Aspect Oriented Programming)`，把`Specification`或`Scenario`当做一个切面，在执行之前和执行之后做一些操作


----------
## Suit Hook
> 作用于所有的`Specification`，在`Specification`执行之前或执行之后执行

```
    //在所有的Specification执行之前执行
    @BeforeSuite
    public void BeforeSuite() {
      // Code for before suite
    }

    //在所有的Specification执行之后执行
    @AfterSuite
    public void AfterSuite() {
      // Code for after suite
    }
```


----------
##Specification Hook
> 作用于`Specification`，可以在某个`Specification`执行之前或执行之后执行

```
    //在每一个Specification执行之前执行
    @BeforeSpec
    public void BeforeSpec() {
      // Code for before spec
    }

    //在每一个Specification执行之后执行
    @AfterSpec
    public void AfterSpec() {
      // Code for after spec
    }
```


----------
##Scenario Hook
> 作用于`Scenario` ，可以在某个`Scenario` 执行之前或执行之后执行

```
    //在每一个Scenario 执行之前执行
    @BeforeScenario
    public void BeforeScenario() {
      // Code for before scenario
    }

    //在每一个Scenario 执行之后执行
    @AfterScenario
    public void AfterScenario() {
      // Code for after scenario
    }
```


----------
##Step Hook
> 作用于`Step`，可以在某个`Step`执行之前或执行之后执行

```
    //在每一个Step执行之前执行
    @BeforeStep
    public void BeforeStep() {
      // Code for before step
    }

    //在每一个Step执行之后执行
    @AfterStep
    public void AfterStep() {
      // Code for after step
    }
```


----------
> `Gauge`默认会在`Scenario`执行之后清除缓存，所以会在下个`Scenario`执行之前创建新的对象，该功能可以在[配置](http://getgauge.io/documentation/user/current/advanced_readings/managing_environments.html#gauge_clear_state_level)中设置清除缓存的等级