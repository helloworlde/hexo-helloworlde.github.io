---
title: Docker 中使用 Dockerfile
date: 2018-04-08 15:21:27
tags:
    - Docker
categories: 
    - Docker
---

# Docker 中使用 Dockerfile

Dockerfile 是一个文件，其包含了一条条的指令（instruction），每一条指令构建一层，因此每一条指令的内容就是描述该层应当如何构建

- 构建一个镜像 

```
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

## 构建

### 命令
#### FROM 
用于指定基础镜像，必备的指令，必须是第一条

- `FROM scratch`
`scratch`是一个特殊的镜像，表示一个空白的镜像，意味着不以任何镜像为基础，接下来的指令作为第一层

#### RUN
`RUN` 指令是用来执行命令的，格式有两种：

- `shell` 格式：`RUN <命令> `

```
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

- `exec`格式：`RUN ["可执行文件", "参数1", "参数2"]`

每一个 `RUN` 命令都会构建一层，应当减少不必要的构建

```
FROM debian:jessie

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

这样会构建7层，会提交大量的无用的改动，应当改为：

```
FROM debian:jessie

RUN buildDeps='gcc libc6-dev make' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
  
```

`Dockerfile` 支持在 Shell 行尾添加 `\`的命令换行方式，以及行首添加 `#` 进行注释

### 执行构建

```
docker build -t myubuntu:v2
```

这样就能构建一个名为`myubuntu`, 标签为`v2`的镜像

- 从 Git Repo 构建 

```
docker build https://github.com/test/test.git#:test
```

这样就会在 `git clone` 之后就会切换到`master`分支，进入到 `test`目录执行构建

- 用压缩包构建 

```
docker build http://server/context.tar.gz
```

#### COPY
用于将构建上下文目录中的源文件复制到新的一层镜像内的目标路径位置 
格式：

```
COPY <源路径> ... <目标路径>
COPY ["<源路径1>", "<目标路径>"]
```

```
COPY package.json /usr/src/app/
COPY home* /mydir
COPY home?.txt /mydir
```

`<目标路径>` 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作路径可以通过`WORKDIR` 指定）

使用`COPY` 指令，源文件的各种源数据都会被保留，比如读、写、执行权限。文件变更时间等

#### ADD 
用于文件复制，和`COPY`一样，但是`<源路径>`可以是个URL，Docker 会将下载链接的文件放到`<目标路径>`中，文件权限为 600，如果`<源路径>`是一个 tar 文件，压缩格式为 `gzip`,`bzip2`,`xz`的情况下会自动解压该文件到`<目标路径>`

```
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /usr/src/
```

#### CMD 
容器启动命令，指令格式和`RUN`相似：
- `shell`格式：`CMD <命令>`
- `exec`格式：`CMD ["可执行文件", "参数1", "参数2", ...]`，这类格式在执行的时候会被解析为`JSON`格数组，因此需要用双引号

```
CMD echo $HOME
CMD ["sh", "-c", "echo $HOME"]

CMD nginx -g daemon off
CMD ["nginx", "-g", "daemon off"]
```

#### ENTRYPOINT
`ENTRYPOINT`的格式和目的和`CMD`一样，都是在指定容器启动程序及参数，需要通过 `docker run --entrypoint`来指定；区别在于 `ENTRYPOINT`可以接收外部命令传入的参数作为内部命令的参数使用

当指定了`ENTRYPOINT`后，`CMD`不再是直接的运行其命令，而是将`CMD`的内容作为参数传给`ENTRYPOINT`

```
<ENTRYPOINT> "<CMD>"
```

#### ENV
用于设置环境变量，格式有两种：

```
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>
```

```
ENV NODE_VERSION 7.2.0
RUN echo $NODE_VERSION
```

`ENV`可以在`ADD`,`COPY`,`ENV`, `EXPOSE`,`LABEL`,`USER`,`WORKDIR`, `VOLUME`,`STOPSIGNAL`, `ONBUILD`

#### ARG
构建参数，和`ENV`的效果一样，都是设置环境变量，但是`ARG`所设置的环境变量在容器运行时不存在
格式：`ATG <参数名>[=<默认值>]`
默认值可以通过`docker build --build-arg <参数名>=<值>`

#### VOLUME 
用于指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据

格式：

```
VOLUME ["<路径1>","<路径2>"]
VOLUME <路径>
```

```
VOLUME /data
```

运行时可以覆盖这个挂载设置：

```
docker run -d -v mydata:/data 
```

这样就使用 `mydata`这个命名卷挂载到 `/data`这个位置，替代了在 `Dockerfile`中定义的匿名卷的挂载配置

#### EXPOSE
用于声明运行时容器提供服务端口，仅仅是一个声明，并不会直接开启端口的服务，用于帮助使用者理解镜像服务的守护端口，同时用于在运行时使用端口随机映射`docker run -P`时使用`EXPOSE`配置的端口

```
EXPOSE <端口1> [<端口2> ...]
```

#### WORKDIR
用来指定工作目录（当前目录），以后各层的当前目录就被改为指定目录，如果目录不存在，会直接生成该目录

格式为：`WORKDIR <工作目录路径>`

#### USER 
`USER`和`WORKDIR`相似，都是改变环境状态并影响以后的层，`WORKDIR`改变的是工作目录，`USER`改变之后执行`RUN`,`CMD`,`ENTRYPOINT`之类命令的身份，`USER`只是切换到指定用户，该用户必须事先建立好

格式：`USER <用户名>`

```
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN ['redis-server']
```

#### HEALTHCHECK
用来告诉 Docker 如何判断容器的状态是否正常
格式：
- `HEALTHCHECK [选项] CMD <命令>`：设置检查容器健康状况的命令
- `HEALTHCHECK NONE`：如果基础镜像有健康检查指令，使用该命令可以屏蔽

`HEALTHCHECK`支持下列选项：
 - `--interval=<间隔>`：两次健康检查的间隔，默认为30s
 - `timeout=<时长>`：健康检查命令运行超时时间，如果超过这个时间则被认为此次健康检查失败
 - `--retries=<时长>`：当连续失败指定次数后，则将容器状态视为`unhealthy`，默认3次

`HEALTHCHECK`只可以出现一次，如果写了多个，则只有最后一个生效；
在`HEALTHCHECK [选项] CMD` 后面的命令，格式和`ENTRYPOINT`一样，分为 `shell`和`exec`格式，命令的返回值决定了改次检查的成功与否，`0`:成功，`1`:失败，`2`: 保留


检查web服务是否可用：

```
FROM nginx
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
HEALTHCHECK --interval=5s --timeout=3s \
CMD curl -fs http://localhost/ || exit 1
```

构建并启动该容器，查看其状态：

```
docker build -t myweb:v1
docker run -d --name web -p 80:80 myweb:v1
```

```
$ docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
03e28eb00bd0 myweb:v1 "nginx -g 'daemon off" 3 seconds ago Up 2 seconds (health: starting) 80/tcp, 443/tcp web

$ docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
03e28eb00bd0 myweb:v1 "nginx -g 'daemon off" 18 seconds ago Up 16 seconds (healthy) 80/tcp, 443/tcp web
```

#### OBUILD 
用于构建下一级镜像时执行，当前镜像并不执行，可用看做通用的构建步骤，在之后的镜像构建中执行

格式 ：`ONBUILD <其他指令>`

构建当前镜像为基础镜像，后续镜像依赖该基础镜像，不需要重复写配置：
当前项目配置：

```
FROM node:slim
RUN mkdir /app
WORKDIR /app
ONBUILD COPY ./package.json /app
ONBUILD RUN [ "npm", "install" ]
ONBUILD COPY . /app/
CMD [ "npm", "start" ]
```

构建

```
docker build -t my-node 
```

其他项目配置：

```
FROM my-node
```

`npm install`,`COPY . /app/`, `npm start`会在后续的每一次构建中都执行