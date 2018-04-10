---
title: Docker 配置 Nginx 访问宿主机目录下的应用
date: 2018-04-10 11:39:09
tags:
    - Docker
    - Ubuntu
    - Tomcat
    - Nginx
categories: 
    - Docker  
    - Ubuntu
    - Tomcat
    - Nginx     
---

# Docker 配置 Nginx 访问宿主机目录下的应用

> 使用 Nginx 将请求转发到宿主机的 Tomcat 应用

## 配置并启动 Tomcat 

## 安装 Docker 

## 配置 Nginx

- 创建配置和日志文件夹

```
mkdir /home/nginx/conf
mkdir /home/nginx/logs
```

- 查询宿主机 IP 

```
docker inspect --format '{{ .NetworkSettings.IPAddress }}' <container-ID> 

# 或
docker inspect <container id> 

# 或
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name_or_id
```

- 添加配置文件 `nginx.conf`

将 `8084`端口转发到`8080`端口，使用 `log_format`目的是为了使用阿里云监控切分日志，可以没有 

```
log_format proxyformat "$remote_addr $request_time $http_x_readtime [$time_local] \"$request_method http://$host$request_uri\" $status $body_bytes_sent \"$http_referer\" \"$upstream_addr\" \"$http_user_agent\" \"$upstream_response_time\" \"$request_time\"";


 server {
      listen 80;
      server_name ali.hellowood.com.cn;
      location / {
        proxy_pass http://172.17.0.1:8080;
        proxy_set_header Host $http_host;                    
        proxy_set_header X-Real-IP $remote_addr;                    
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
      }
  }  
```

> 需要注意的是，Docker 会默认使用桥接路由，所以其 IP 和宿主机的 IP 位于同一 IP 段，并且默认宿主机的 IP 为第一个，即如果 Docker 的 IP 为 `10.1.0.2`, 则可以通过 访问 `10.1.0.1` 访问到宿主机 

- 拉取 Nginx 镜像

```bash
docker pull nginx
```

- 启动容器

```
docker run --name nginx -d -p 8084:80 -v /home/nginx/conf:/etc/nginx/conf.d -v /home/nginx/logs:/var/log/nginx nginx
```

待容器启动后访问`http://ali.hellowood.com.cn:8084`就可以看到 Tomcat 的页面了