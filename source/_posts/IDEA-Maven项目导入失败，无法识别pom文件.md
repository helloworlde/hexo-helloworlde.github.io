---
title: IDEA Maven项目导入失败，无法识别pom文件
date: 2018-01-01 11:23:33
tags:
    - Java
    - Maven 
    - IDEA
    - Experience
categories: 
    - Java
    - Maven
    - IDEA
    - Experience
---
## 解决思路
> #### 按照以下顺序逐个检查，更改host文件比价极端，应该很少见
#### 1. 重启IDEA
#### 2. 重启电脑
#### 3. 重新导入项目
#### 4. 重装Maven
#### 5. 重装IDEA
#### 6. 检查host文件中有没有其他地址指向`localhost`

-----------

> 一个Maven项目，之前是可以正常使用的，没有任何问题，但是今天去Debug启动Tomcat，提示Socket被占用， 错误信息如下：

```
Error running 'Console': Unable to open debugger port (127.0.0.1:63347): jav...
```
> - 正常启动时又提示JVM的1099端口被占用，无论如何修改端口都无效

> - 先看到了这个帖子[nable to open debugger port (127.0.0.1:63777): java.net.BindException "Address already in use: JVM](http://blog.csdn.net/lutinghuan/article/details/45693577)，之后重启IDEA，重启电脑，重新导入项目，最后重装IDEA依然不好用；
> - 之后又觉得可能是Maven的问题，运行Maven的命令后发现Maven是正常的；
> - 最后发现了[IntellijIDEA 无法创建Maven工程，导入已有工程无法识别pom文件](http://www.jianshu.com/p/fb7bddca7b1e)这篇博客，发现问题挺像的，而我确实改过host文件，**将内网IP指向了`localhost`**；改回来之后发现可以正常导入了

> - 问题的原因应该就是因为改了host文件导致localhost无法正常访问，所有各种端口都不行，也导致了无法访问Maven仓库，所以IDEA无法识别项目破，pom文件
### host文件除了`127.0.0.1`和`::1`之外不要有其他的地址指向`localhost`