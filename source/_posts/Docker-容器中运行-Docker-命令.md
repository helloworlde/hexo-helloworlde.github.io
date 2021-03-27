---
title: Docker 容器中运行 Docker 命令
date: 2018-08-08 16:04:24
tags:
    - Docker
categories: 
    - Docker  
---

# Docker 容器中运行 Docker 命令

> 在使用 GitLab/Jenkins 等 CI 软件的时候需要使用 Docker 命令来构建镜像，需要在容器中使用 Docker 命令；通过将宿主机的 Docker 共享给容器即可

- 在启动容器时添加以下命令：

```bash
--privileged \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $(which docker)r:/bin/docker \
```

> - `--privileged` 表示该容器真正启用 root 权限
> - `-v /var/run/docker.sock:/var/run/docker.sock`和`-v $(which docker)r:/bin/docker`命令将相关的 Docker 文件挂载到容器

-----------------------------

- Demo: 启动 GitLab

```bash
docker run --name gitlab-ee \
    -d -p 443:443 -p 80:80 -p 22:22 \
    --privileged \
    --restart always \
    --hostname 10.0.0.24 \
    -v /Users/hellowood/gitlab/logs:/var/log/gitlab \
    -v /Users/hellowood/gitlab/data:/var/opt/gitlab \
    -v /Users/hellowood/.m2:/root/.m2 \
    -v /Users/hellowood/.gradle:/root/.gradle \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $(which docker):/bin/docker \
    gitlab/gitlab-ee:latest
```
