---
title: Gauge基础知识
date: 2018-01-01 11:29:04
tags:
    - Java
    - Gauge
    - Test
categories: 
    - Java
    - Gauge
    - Test
---
本文所有内容均参照自[Gauge官方文档](http://getgauge.io/documentation/user/current/)

----------
>#基本思想
>`Gauge`入门比较简单，`Gauge`的基本思想就是通过`.spec` 或者`.md` 文件，使用`MarkDown`语法去规定执行的动作，然后由`Java`或者其他语言的文件去按照所写的`.spec` 或者`.md` 文件的顺序去执行`Java`文件，从而达到测试的目的


----------
# 简单语法


----------


## Specification
> - 作用：开始标志，只能有一个，每个Specification至少包含一个Scenario

```
Specification name          
==================

或者：  

# Specification name
```


----------


## Scenario
> - 作用：特定的场景中的一个情节，一个或多个Scenario组成一个Specification，每个Scenario至少包含一个Step

```
Scenario name                 
-------------

或者：

## Scenario name

```


----------


## Step
> 作用：Specification的可执行部分

```
* Step Name
```



> -  一般Step
    -  正常执行的Step，包含在Scenario中
```
    * step
```

 
> - Context Step 
     -  在Scenario执行之前执行的操作，在每个Scenario执行之前都会先执行Context Step
```
    * Context step
```

> - Teardown Step
     -  在Scenario执行之后执行的操作，在每个Scenario执行之后都会执行 Teardown Step
     -用半角下划线标识，不是横线
  
        ________________
        Teardown Step
        
        * Teardown Step1
        * Teardown Step2

```

    Delete project
    ==============
    
    * Sign up for user "mike"
    * Log in as "mike"
    
    Delete single project
    ---------------------
    * Delete the "example" project
    * Ensure "example" project has been deleted
    
    Delete multiple projects
    ------------------------
    * Delete all the projects in the list
    * Ensure project list is empty
    
    ____________________
    These are teardown steps
    
    * Logout user "mike"
    * Delete user "mike"
```
> 执行步骤：

> 1. Context steps execution
> 2. `Delete single project scenario execution`
> 3. Tear down steps execution
> 4. Context steps execution
> 5. `Delete multiple projects scenario execution`
> 6. Tear down steps execution

----------
## Tags
> - 作用：用于标记Specification 和 Scenario


```
    Specification 
    ===================          
    Tags: spec, login

      
    Scenario 
    --------------
    Tags: scenario, main-page

```


----------
## Concept
> - 作用：可重用的逻辑组成的单元，写在单独的文件中用于多次使用


```
在.spec文件中：
    

```
    * Delete product "Learning Go"
    
```

在.cpt文件中：

    # Delete product <name>

    * Find and Open product page for <name>
    * Delete this product   
```


----------

## Parameters
 - 作用：将参数传递给Java或其他文件

```
    * Sign up for user "mike"
```

 - 静态参数
  - 使用 `"param"`形式
```
    * Check "product" exists
```

 -  动态参数
   - 使用`<param>`形式

```
    * Check <product> exists
```
- Table参数

```
    |id| name |
    |--|------|
    |1 | john |
    |2 | mike |

```

 - 特殊参数
  -  使用`<prefix:value>`形式
      - prefix ： 参数类型，可以是file，table等
      - value：参数值
  
```
File：

    * Verify email text is <file:email.txt>
    * Check if <file:/work/content.txt> is visible
    
CSV：
    * Step that takes a table <table:resources/user.csv>
    * Check if the following users exist <table : /Users/john/work/users.csv>
```

- resources/user.csv

```
FredFlintstone
JohnnyQuest
ScroogeMcduck

```

> 路径可以使相对或者绝对路径

----------
##Comments
 - 作用：备注信息

```
    This is comments!!!
```
----------
##Images
 - 作用：

```
    ![Alt text](/path/to/img.jpg)

    ![Alt text](/path/to/img.jpg "可选的标题")
```

> Image路径应使用相对路径
    
 ----------
##Links
 - 作用：

```
    This is [an example](http://getgauge.io "Title") inline link.

    [This link](http://github.com/getgauge/gauge) has no title attribute.
```
