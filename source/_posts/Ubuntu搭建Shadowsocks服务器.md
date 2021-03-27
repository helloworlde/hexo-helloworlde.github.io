---
title: Ubuntu搭建Shadowsocks服务器
date: 2018-01-01 00:58:24
tags:
    - Ubuntu
    - Shadowsocks 
categories: 
    - Ubuntu
    - Shadowsocks 
---

> 在Ubuntu环境中搭建Shadowsocks服务

##1 安装Shadowsocks

```
# 更新apt-get
sudo apt-get update

# 安装python包管理工具
sudo apt-get install python-pip

# 安装Shadowsocks
pip install shadowsocks
```

##2 配置Shadowsocks

- 创建配置文件

```
mkdir shadowsocks
vi config.json
```
- 在config.json文件中添加以下配置

```
{
  "server": "0.0.0.0",
  "server_port": 8623,
  "local_address": "127.0.0.1",
  "local_port": 1080,
  "password": "your password",
  "timeout": 300,
  "method": "aes-256-cfb",
  "fast_open": false
}
```
- 配置说明

| 字段 | 说明|
|----|----|
|server|服务端监听地址|
|server_port|服务端端口|
|local_address|本地监听地址|
|local_port|本地监听端口|
|password|密码|
|timeout|超时时间（秒）|
|method|加密方法|
|fast_open|是否启用TCP-Fast-Open，true或者false|


##3 启动Shadowsocks

- 启动：

```
 sudo ssserver -c /home/ubuntu/develop/shadowsocks/shadowsocks.json -d start
```


- 停止：

```
 sudo ssserver -c /home/ubuntu/develop/shadowsocks/shadowsocks.json -d stop
```


- 重启：

```
 sudo ssserver -c /home/ubuntu/develop/shadowsocks/shadowsocks.json -d restart
```

##4 设置开机自启动

```
sudo vi /etc/rc.local
```
- 加入以下内容

```
sudo ssserver -c /home/ubuntu/develop/shadowsocks/shadowsocks.json -d start
```

## 5  配置多个用户
- 将配置文件修改为以下：

```
{
  "server": "0.0.0.0",
  "local_address": "127.0.0.1",
  "local_port": 1080,
  "port_password": {
    "8623": "your password1",
    "8624": "your password1",
    "8625": "your password2",
    "8626": "your password3",
    "8627": "your password4"
  },
  "timeout": 300,
  "method": "aes-256-cfb",
  "fast_open": false
}
```