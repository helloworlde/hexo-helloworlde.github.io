---
title: Docker 使用 NFS 作为数据卷存储
date: 2022-9-22 11:20:19
tags:
- Docker
- NFS
- Ubuntu
- HomeLab
categories:
- HomeLab
---

# Docker 使用 NFS 作为数据卷存储

在搭建 HomeLab 的过程中，出现因虚拟机故障导致容器挂载在虚拟机上的数据丢失的问题，因此，将数据挂载在虚拟机上仍然存在风险；同时为了将计算和数据分离，HomeLab 所在的服务器只做计算，将数据存储转移到 NAS上；因此，使用 NFS 作为 Docker 的数据卷，将数据挂载到远程的 NAS 存储中

Docker 支持 Samba/NFS 等协议的远程存储

## 创建 Docker NFS 数据卷

通过 docker 命令创建 NFS 的数据卷，在创建时，指定驱动为 `local`，类型是 `nfs`，同时指定地址和协议版本，以及服务端的挂载路径，名称为 `nginx-volume`

```bash
docker volume create --driver local \
--opt type=nfs \
--opt o=addr=192.168.2.10,nolock,vers=4,soft,rw \
--opt device=:/workspaces/data/docker/nginx \
nginx-volume
```

查看 `nginx-volume` 信息

```bash
docker volume inspect nginx-volume

[
    {
        "CreatedAt": "2022-09-22T14:48:26+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/nginx-volume/_data",
        "Name": "nfs-volume",
        "Options": {
            "device": ":/workspaces/data/docker/nginx",
            "o": "addr=192.168.2.10,nolock,vers=4,soft,rw",
            "type": "nfs"
        },
        "Scope": "local"
    }
]
```

## 容器中使用 NFS 数据卷

### 在容器中挂载

以 Nginx 为例，挂载 `nginx-volume` 到容器中；也可以使用 `-v nginx-volume:/data`的方式挂载，这两个命令的区别在于如果挂载的数据卷不存在，`-v` 会创建一个，而 `--mount`会报错

```bash
docker run -d -it \
--name nginx \
-p 80:80 \
--mount source=nginx-volume,target=/data\
nginx
```

启动后进入容器的 `/data` 目录，发现存在 `test.txt` 文件，说明挂载成功

```bash
docker exec -it nginx bash

ls /data
test.txt
```

### 在 docker-compose 中挂载

- 挂载已创建的数据卷

```yaml
version: '3'

services:
nginx:
image: nginx
container_name: nginx
hostname: nginx
ports:
- 80:80
volumes:
- nginx-volume:/data

volumes:
nginx-volume:
external: true
```

- 创建新的数据卷

```yaml
version: '3'

services:
nginx:
image: nginx
container_name: nginx
hostname: nginx
ports:
- 80:80
volumes:
- nginx-volume:/data

volumes:
nginx-volume:
driver_opts:
type: "nfs"
o: "addr=192.168.2.10,nolock,vers=4,soft,rw"
device: ":/workspaces/data/docker/nginx"
```
