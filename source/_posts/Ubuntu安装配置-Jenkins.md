---
title: Ubuntu安装配置 Jenkins
date: 2018-01-01 12:12:32
tags:
    - Jenkins 
    - Ubuntu
categories: 
    - Jenkins
    - Ubuntu
---
> 在 Ubuntu 下 Jenkins 服务

## 1. 安装
- 安装前确认JDK环境配置是正确的
```
wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

sudo apt-get update
sudo apt-get install jenkins

```
> 等待安装完成，如果默认的8080端口没有被占用，则会自动启动 Jenkins 服务， 如果有其他程序占用了 8080 端口，则 Jenkins 会启动失败
##2. 更改 Jenkins 端口
> 需要更改两个文件

- 修改`/etc/default/jenkins`文件，将 `HTTP_PORT=8080`更改为你需要的端口
- 修改`/etc/init.d/jenkins`文件，将`do_start`函数的`check_tcp_port`命令改为需要的端口

##3. 启动 Jenkins 服务

```
sudo /etc/init.d/jenkins start
```

##4. 访问并配置
> 访问 Jenkins 的地址，会要求填写 Jenkins 的密码，该密码可以在`/var/log/jenkins/jenkins.log`中找到