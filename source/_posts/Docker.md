---
title: Docker
date: 2018-04-08 15:18:49
tags:   
    - Docker 
categories: 
    - Docker
---

# Docker

## Docker 中的概念
 - 镜像：一个特殊的文件系统，提供容器运行时所需的程序，库，资源，配置和配置参数，不包含任何动态数据，内容在构建之后也不会被改变
 - 容器： 镜像和容器可以看做是面向对象中的类和实例，容器的实质是进程，运行于一个隔离的环境，容器运行时，已当前镜像为基础，在其上创建一个当前容器的存储层，容器消亡时，任何存储于容器存储层的数据也会被被删除
 - 仓库：集中存储，分发镜像的服务


## 安装
####  Ubuntu：
- 卸载旧版本

```
sudo apt-get remove docker \
               docker-engine \
               docker.io
```

- 安装 Docker CE

```
sudo apt-get update
sudo apt-get install docker-ce
```

- 或使用脚本安装

```
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh --mirror Aliyun
```

- 启动 Docker CE

```
sudo systemctl enable docker
sudo systemctl start docker
```

- 添加 Docker 组，并将当前用户加入 Docker 组

```
sudo groupadd docker
sudo usermod -aG docker $USER
```

- 测试

```
docker run hello-world
```

- 镜像加速 
编辑或新建 `/etc/docker/daemon.json` ，添加以下内容

```
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```

- 重新启动

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### Mac：
- 安装

```
brew cask install docker
```

- 启动 
点击 Docker 图标启动

- 测试

```
docker version
docker info
```

- 镜像加速
在任务栏点击 Docker 图标 -> Perferences... -> Daemon -> Registry mirrors， 添加以下 URL 后重新启动

```
https://registry.docker-cn.com/
```

## 使用

### 拉取镜像

```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```
Docker 镜像仓库地址：地址格式为 <域名/IP>[:端口号]
仓库名：<用户名>/<软件名>，用户名如果不写，则默认为 `library`，即官方镜像

```
docker pull ubuntu:16.04
```

### 运行

```
docker run -it --rm ubuntu:16.04 bash 
```

- `-it`: `-i`:进入交互操作，`-t`:进入终端
- `--rm`:容器退出后即删除
- `ubuntu:16.04`:使用 `ubuntu:16.04`作为基础镜像启动
- `bash`: 放在镜像名后的是命令，进入交互式 `Shell` 使用 `bash`

### 列出镜像

```
docker image ls
```
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              e548f1a579cf        11 days ago         109MB
ubuntu              16.04               0458a4468cbc        5 weeks ago         112MB
```

分别是`仓库名`，`标签`，`镜像ID`，`创建时间`，`所占用的空间`

### 删除镜像

```
docker image rm [选项]<镜像1> [<镜像2>...]
```

```
// 删除名为 ubuntu 的镜像
docker image rm ubuntu
// 删除名为 ubuntu，标签为 16.04 的镜像
docker image rm ubuntu:16.04
// 删除所有仓库名为 redis 的所有镜像
docker image rm $(docker image ls -q redis)
// 删除所有在 `mongo:3.2`之前的镜像
docker image rm $(docker image ls -q -f before=mongo:3.2)
```

- `Untagged` 和 `Deleted`
`Untagged` 取消标签，当删除镜像时先取消标签，当该镜像的所有标签都被取消且没有其他的镜像依赖该镜像时才会执行 `Deleted`，如果有其他的标签指向该镜像或被其他镜像依赖，则不会删除

### Commit 镜像

```
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]
```

```
docker run --name webserver -d -p 80:80 nginx
```

做一些改动之后提交

```
docker exec -it webserver bash
echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
exit
 
docker commit --author "HelloWood" --message "修改首页" webserver nginx:v2
```