---
title: Windows Server搭建Shadowsocks服务器
date: 2018-01-01 12:18:00
tags:
    - Shadowsocks 
categories: 
    - Shadowsocks 
---
> 在Windows Server 下搭建Shadowsocks，使用最简单的方式搭建

## 1. 下载Shadowsocks程序
下载并解压Shadowsocks文件[https://github.com/shadowsocks/libQtShadowsocks/releases](https://github.com/shadowsocks/libQtShadowsocks/releases)

## 2. 添加config.json配置文件
```
{
    "server":"localhost",        //配置服务端地址，不需要修改
    "server_port":623,           //服务端端口，可以根据需要修改，建议改成大点的不会被占用的端口
    "local_address":"127.0.0.1", //本地地址，不需要修改
    "local_port":1080,           //本地端口，可以根据需要修改
    "password":"Orange",         //密码
    "timeout":600,               //连接超时时间
    "method":"rc4-md5",          //加密方式
    "http_proxy": false,         //代理
    "auth": false                //需要验证
}
```

## 3. 运行shadowsocks-libqss.exe启动服务
有两种方式启动服务：

- 在命令行中运行以下命令

```
shadowsocks-libqss.exe -c config.json -S
```
- 将命令写在.bat文件中，运行.bat文件启动
      
```
    @echo off
    shadowsocks-libqss.exe -c config.json -S
```


> 配置完成，在客户端配置后就可以使用了

下载配置文件
- 参考链接： 
    - Shadowsocks服务端：[https://github.com/shadowsocks/libQtShadowsocks/releases](https://github.com/shadowsocks/libQtShadowsocks/releases)
    - Shadowsocks客户端：[https://github.com/shadowsocks/shadowsocks-windows/releases](https://github.com/shadowsocks/shadowsocks-windows/releases)