---
title: Gauge中执行测试的常用命令
date: 2018-01-01 11:30:51
tags:
    - Java
    - Gauge
    - Test
categories: 
    - Java
    - Gauge
    - Test
---
本文所有内容均参考自Gauge官方文档s

-----------

#初始化`Java`项目
> 会在指定的文件夹下创建一个新的`Gauge`项目，如果没有安装`Java`和`Html-report`插件会自动安装

```
    gauge --init java
```


----------

# 执行项目
## 通过文件执行
- 执行`specs`文件夹下所有`.spec`文件

```
    gauge specs
```
- 执行`specs`文件夹下的`specs.spec`文件

```
    gauge specs/specs.spec
```
- 执行多个文件夹下的所有文件

```
    gauge specs-dir1/ specs-dir2/
```
- 执行多个文件夹下的指定文件

```
    gauge specs-dir1/specs1.spec specs-dir2/spesc2.spec
```
- 执行一个特定的`Scenario`

```
    gauge specs/specs.spec:16
```
> 后面的数字是要执行的`Scenario`所在的行数，从0开始

- 执行多个特定的`Scenario`

```
    gauge specs-dir1/specs1.spec:10 specs-dir2/specs2.spce:20
```
- 执行的过程中输出日志

```
    gauge --verbose specs
```


----------


##通过`Tag`执行

```
    Login specification
    ===================
     Tags: login, admin, user login
    
    
    Successful login scenario
    -------------------------
     Tags: login-success, admin
 
 
     failed login scenario
    -------------------------
     Tags: login-failed, admin
```
- 通过单独的`Tag`执行

```
    gauge --tags admin specs
```
> 带有admin 的所有的`Specification`或`Scenario`都会被执行

- 通过多个`Tag`执行

```
    gauge --tags login,admin specs

    或者：
    
    gauge --tags "login,admin" specs

    或者：

    gauge --tags "login & admin" specs
```
> 只有当`Specification`或者`Scenario`同时有`login`和`admin`  Tag的时候才会被执行

- 执行含有空格的`Tag`
```
    gauge --tags "user login" specs
```
- Tag支持`与、或、非` 运算

```
    !TagA: 执行不含有TagA的Specification或Scenario
    TagA & TagB: 执行同时含有TagA和TagB的Specification或Scenario
    TagA & !TagB: 执行含有TagA但不含TagB的Specification或Scenario
    TagA | TagB: 执行含有TagA或TagB的Specification或Scenario
    (TagA & TagB) | TagC: 执行同时含有TagA和TagB或者含有TagC的Specification或Scenario
    !(TagA & TagB) | TagC: 执行同时不含有TagA和TagB的或者含有TagC的Specification或Scenario
    (TagA | TagB) & TagC: 执行同时含有TagA和TagC或者TagB和TagC的Specification或Scenario

```

----------
##并发执行
> 并发执行是根据CPU核心数量创建多个流来提高执行速度，均衡负载

- 使用并发执行
```
    gauge --parallel specs

    或者：

    gauge -p specs
```
- 指定并发执行的流的数量

```
    gauge --parallel -n=4 specs
```
- 延迟分配策略
> 在执行期间根据执行状态动态的分配执行任务，每个流执行的`Specification`数量不一定相同

```
    gauge -n=4 --stategy="lazy" specs

    或者：
    
    gauge -n=4 specs
```
- 积极加载策略
> 在执行之前平均的分配相同数量的`Specification`给每个流

```
    gauge -n=4 --strategy="eager" specs
```
- 分组执行策略
> 对`Specifications` 按照字母表的顺序排序后分组，可以指定某一组执行

```
    gauge -n=4 -g=2 specs
```
> 把`specs`文件夹下的所有.`spec` 文件按字母表的顺序分成4组，只执行第二组


----------

## 使用[钩子(Hook)](http://getgauge.io/documentation/user/current/advanced_readings/execution_hooks/context.html)来查看执行状态
> 钩子可以理解为`Java`中的`AOP(Aspect Oriented Programming)`，把`Specification`或`Scenario`当做一个切面，在执行之前和执行之后做一些操作

- 根据上下文执行
```
    public class ExecutionHooks {

        @BeforeScenario
        public void loginUser(ExecutionContext context) {
          String scenarioName = context.getCurrentScenario().getName();
          // Code for before scenario
        }
    
        @AfterSpec
        public void performAfterSpec(ExecutionContext context) {
          Specification currentSpecification = context.getCurrentSpecification();
          // Code for after step
        }
}
```
- 根据`Tag`标记来执行

```
    // 在含有tag1和tag2的Specification或Scenario之前执行
    @BeforeSpec(tags = {"tag1, tag2"})
    public void loginUser() {
        // Code for before scenario
    }
    
    
    // 在含有tag1或tag2的Specification或者Scenario执行之后执行
    // Tag集合(tagAggregation)的运算符是或
    @AfterStep(tags = {"tag1", "tag2"}, tagAggregation = Operator.OR)
    public void performAfterStep() {
        // Code for after step
    }
```

