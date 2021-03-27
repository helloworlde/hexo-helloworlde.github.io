---
title: Ubuntu/Docker 替换软件源
date: 2019-05-27 15:06:43
tags:
    - Docker
    - Ubuntu
categories: 
    - Docker
    - Ubuntu
---

# Ubuntu/Docker 替换软件源

## Ubuntu 

### 使用 sed 命令

```bash
sudo sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apt/sources.list/
```

### 手动修改

```bash
cd /etc/apt/

sudo mv sources.list sources.list.bak

sudo vi sources.list
```

输入以下内容

```
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse  
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse  
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse  
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse  

deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse  

deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse  

deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse  

deb http://archive.canonical.com/ubuntu/ xenial partner  
deb http://extras.ubuntu.com/ubuntu/ xenial main  
```

```bash
sudo apt-get update
sudo apt-get upgrade
```

## Docker alpine 镜像替换软件源

### 在 Dockerfile 中添加

```dockerfile
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
```

### 修改容器中软件源

#### 使用 sed 命令

```bash
sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
```

### 手动修改

```bash
vi etc/apk/repositories
```
输入以下内容

```
http://mirrors.aliyun.com/alpine/latest-stable/main/
http://mirrors.aliyun.com/alpine/latest-stable/community/
```

----------

### 参考文档(包含其他平台的修改)

- [USTC Mirror Help](https://mirrors.ustc.edu.cn/help/)