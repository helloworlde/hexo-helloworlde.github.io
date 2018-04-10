---
title: Ubuntu安装 Redis -编译方式安装
date: 2018-01-01 12:09:09
tags:
    - Ubuntu
    - Reids 
categories: 
    - Ubuntu
    - Reids 
---
> 使用 `sudo apt install redis-server`安装的 Redis 并不是最新的，通过在官网下载来安装最新的 Redis

## 1 创建文件夹

```
mkdir redis
```

## 2 下载 Redis

```
wget http://download.redis.io/releases/redis-4.0.1.tar.gz
```

## 3 解压

```
 tar -xzvf redis-4.0.1.tar.gz
```

## 4 编译安装

```
cd redis-4.0.1
make 
make test
sudo make install 
```
## 5 配置
- 设置登录密码

```
# 取消注释requirepass
requirepass your_password
```

- 取消登录IP限制

```
# 注释bind
# bind 127.0.0.1
```

## 6 启动

```
redis-server ./redis-4.0.1/redis.conf
```