---
title: Docker 常用命令
date: 2018-04-08 15:20:02
tags:
    - Docker
categories: 
    - Docker
---
#Docker 常用命令 

- run
新建并启动容器

```
// 启动并输出 Hello World
docker run ubuntu:14.04 /bin/echo 'Hello World'

// 启动 bash 终端
docker run -t -i ubuntu:14.04 bash


// 启动一个已终止的容器
docker container start myubuntu

docker run -d ubuntu:14.04 -c 
```

`-t` 启动终端并绑定到容器标准输入上
`-i` 保持容器标准输入打开
`-c` 将容器的输出信息输出到宿主机
`-d` 不会将容器的输出输出到宿主机

- stop
停止容器

```
docker container stop myubuntu
```

- restart 
重启容器

```
docker container restart myubuntu
```

- attach
进入容器，可以执行命令

```
docker container start myubuntu
docker attach myubuntu
```

此时执行`exit`会退出容器

- exec 
进入容器，可以执行命令，需要带参数
`-i`:由于没有分配伪终端，所以不会有命令提示符，但是命令执行结果依然可以返回
`-i -t`:可以显示终端

```
docker container start myubuntu
docker exec -i myubuntu bash
docker exec -it myubuntu bash
```

此时执行`exit`不会退出容器


- save 
将镜像保存为归档文件

```
docker save ubuntu | gzip > ubuntu-last.tar.gz
```

- load 

加载`save`保存的镜像

```
docker load -i ubuntu-last.tar.gz
```

迁移镜像：

```
docker save <镜像名> | bzip2 | pv | ssh <用户名>@<主机名> 'cat | docker load'
```

- export
导出容器快照到本地

```
docker export myubuntu > myubuntu.tar
```

- import 
导入容器快照到本地

```
// 从网络导入
docker import  http://example.com/exampleimage.tgz example/imagerepo

// 从本地导入
cat myubuntu.tar | docker import - myubuntu:v1.0
```

- rm 
删除容器或者镜像

```
docker image rm myubuntu:v1.0

docker container rm myubuntu
```

- prune
删除所有处于终止状态的容器

```
docker container prune
```