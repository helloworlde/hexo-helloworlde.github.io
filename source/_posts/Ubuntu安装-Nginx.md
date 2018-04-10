---
title: Ubuntu安装 Nginx
date: 2018-01-01 12:10:38
tags:
    - Nginx 
    - Ubuntu
categories: 
    - Nginx
    - Ubuntu
---
> 在 Ubuntu 上安装 Nginx

# 1. 通过 Ubuntu 的仓库安装

## 安装

- 安装

```
    sudo apt-get install nginx
```

- 目录结构
    - 所有文件都在 `/etc/nginx/`目录下
    - 程序文件在`/user/local/nginx/sbin/`目录下 
    - 日志在`/var/log/nginx/`目录下
    - 启动脚本为`/etc/init.d/nginx`

## 启动

```
    sudo /etc/init.d/nginx start
```

-----------------------------

# Nginx 源代码安装

## 安装

- 下载

```
    cd /usr/local 
    //下载源码
    wget http://nginx.org/download/nginx-1.2.8.tar.gz

    // 解压
    tar -zxcf nginx-1.2.8.tar.gz
```

- 安装依赖

```
    sudo apt-get install libpcre3
    sudo apt-get install libpcre3-dev
    sudo apt-get install zlib1g-dev
```

- 编译安装

```
    cd nginx-1.2.8
    // 检查环境
    ./configure
        
    // 编译
    make

    // 安装
    make install
```

## 配置
 
 - 默认启动

```
    cd /usr/local/nginx

    // 启动
    sbin/nginx
```

- 配置快捷启动方式

```
    // 创建启动脚本
    sudo vi /etc/init.d/nginx

    // 增加执行权限
    sudo chmod a+x /etc/init.d/nginx

    // 启动
    sudo /etc/init.d/nginx start
    // 停止
    sudo /etc/init.d/nginx stop
    // 重启
    sudo /etc/init.d/nginx restart
```