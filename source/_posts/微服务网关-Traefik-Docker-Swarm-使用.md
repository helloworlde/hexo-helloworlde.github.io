---
title: 微服务网关 Traefik - Docker Swarm 使用
date: 2019-03-24 22:08:30
tags:
    - Traefix 
    - Docker
categories: 
    - Traefix 
    - Docker
---

# 微服务网关 Traefik - Docker Swarm 使用

[traefik](https://docs.traefik.io/) 是一个用 Go 开发的适用于微服务的反向代理和负载均衡的网关；可以自动发现并代理服务，可以用 Kubernetes 或 Docker Swarm 等方式，支持使用 Eureka，Consul，Etcd，ZooKeeper 等注册中心

## Docker Swarm 使用

### 启动官方 Demo

- docker-compose.yml

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik # The official Traefik docker image
    command: --api --docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - "80:80"     # The HTTP port
      - "8080:8080" # The Web UI (enabled by --api)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
  whoami:
    image: containous/whoami # A container that exposes an API to show its IP address
    labels:
      - "traefik.frontend.rule=Host:whoami.docker.localhost"
```
- 启动
```
docker-compose up -d
```

这样会启动一个 treafik 的 WebUI 和一个后端服务器

![01](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/treafik-01.png)

![02](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/treafik-02.png)

浏览器访问 [http://whoami.docker.localhost](http://whoami.docker.localhost)，会返回以下内容 

```
Hostname: 460d64ac7842
IP: 127.0.0.1
IP: 172.22.0.3
GET / HTTP/1.1
Host: whoami.docker.localhost
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.121 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh,zh-CN;q=0.9,en;q=0.8,en-US;q=0.7
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 172.22.0.1
X-Forwarded-Host: whoami.docker.localhost
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 2d620da469b5
X-Real-Ip: 172.22.0.1
```

#### 修改访问规则，使用后缀访问

修改 docker-compose.yml中whoami的访问规则

```
version: '3'

services:
  reverse-proxy:
    image: traefik # The official Traefik docker image
    command: --api --docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - "80:80"     # The HTTP port
      - "8080:8080" # The Web UI (enabled by --api)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
  whoami:
    image: containous/whoami # A container that exposes an API to show its IP address
    labels:
      - "traefik.frontend.rule=PathPrefixStrip:/whoami"
```

再次启动

```
docker-compose up -d 
```

通过浏览器访问 [http://localhost/whoami](http://localhost/whoami) 会输出以下内容：
```
Hostname: 9fc59eb66eca
IP: 127.0.0.1
IP: 172.22.0.3
GET / HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.121 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh,zh-CN;q=0.9,en;q=0.8,en-US;q=0.7
Cookie: XSRF-TOKEN=54b4c46c-68cb-4f05-a8f7-9a2c43aa74e3; JSESSIONID=4A5969048B72F3A0F0122CA901C60445
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 172.22.0.1
X-Forwarded-Host: localhost
X-Forwarded-Port: 80
X-Forwarded-Prefix: /whoami
X-Forwarded-Proto: http
X-Forwarded-Server: 2d620da469b5
X-Real-Ip: 172.22.0.1
```

#### 指定域名访问

- 修改 docker-compose.yml 

```
version: '3'

services:
  reverse-proxy:
    image: traefik # The official Traefik docker image
    command: --api --docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - "80:80"     # The HTTP port
      - "8080:8080" # The Web UI (enabled by --api)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
  whoami:
    image: containous/whoami # A container that exposes an API to show its IP address
    labels:
      - "traefik.frontend.rule=Host:whoami.localhost"
```

- 同时使用域名和后缀

```
version: '3'

services:
  reverse-proxy:
    image: traefik # The official Traefik docker image
    command: --api --docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - "80:80"     # The HTTP port
      - "8080:8080" # The Web UI (enabled by --api)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
  whoami:
    image: containous/whoami # A container that exposes an API to show its IP address
    labels:
      - "traefik.frontend.rule=Host:whoami.docker.localhost"
      - "traefik.frontend.rule=PathPrefixStrip:/whoami"
```

也可以指定 Header，Host 和 Path 的正则或者其他各种访问规则，具体可以参考官方文档[Matchers](https://docs.traefik.io/basics/#matchers)

### 部署单个应用

- 部署一个使用 MongoDB 作为存储的单体应用

docker-compose.yml

```yaml
version: '3'

services:
  reverse-proxy:
    image: traefik # The official Traefik docker image
    command: --api --docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - "80:80"     # The HTTP port
      - "8080:8080" # The Web UI (enabled by --api)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
  mongo: 
    image: mongo
    ports: 
      - "27017:27017"
  angular:
    image: hellowoodes/angular
    depends_on: 
      - reverse-proxy
      - mongo
    expose:
      - "8081"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    labels:
      - "traefik.frontend.rule=Host:angular.localhost"
      - "traefik.frontend.rule=PathPrefixStrip:/"
      - "traefik.backend=angular"
      - "traefik.port=8081"
      - "traefik.weight=10"
      - "traefik.enable=true"
      - "traefik.passHostHeader=true"
      - "traefik.docker.network=ntw_front"
      - "traefik.frontend.entryPoints=http"
      - "traefik.backend.loadbalancer.swarm=true"
      - "traefik.backend.loadbalancer.method=drr"
```

- 启动

```bash
docker-compose up -d
```

启动后访问 [http://angular.localhost](http://angular.localhost/)

![03](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/treafik-03.png)