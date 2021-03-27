---
title: Ubuntu安装JDK
date: 2018-01-01 12:11:39
tags:
    - Ubuntu
    - JDK 
categories: 
    - Ubuntu
    - JDK 
---
# 在Ubuntu通过Java官网下载安装JDK

---------------------

## 安装
1  创建文件夹

```
mkdir java
```

2  下载JDK
> JDK 下载地址[http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
```
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u144-b01/090f390dda5b47b9b721c7dfaa008135/jdk-8u144-linux-x64.tar.gz
```

3  解压

```
 tar -zxvf jdk-8u144-linux-x64.tar.gz
```

4  配置环境变量

- 编辑配置文件
```
sudo vim ~/.bash_profile
```
- 输入以下内容

```
export JAVA_HOME=/home/ubuntu/java/jdk1.8.0_144
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```
- 使配置生效

```
source ~/.bash_profile
```
5  检查是否配置正确

```
java -version
```
- 出现以下内容：
```
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
ubuntu@local:~/java/jdk1.8.0_144$
```

6  查看JDK的路径

```
whereis java
which java （java执行路径）
echo $JAVA_HOME
echo $PATH
```