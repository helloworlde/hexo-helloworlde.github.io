---
title: Nginx 配置
date: 2018-01-01 12:03:12
tags:
    - Nginx 
categories: 
    - Nginx
---
## 1. 基础配置

```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    server {
       listen       80;
        server_name  project.hellowood.com;

        access_log  logs/project.access.log;

        location / {
                proxy_pass http://127.0.0.1:8080;
                proxy_set_header   Host    $host;  
                proxy_set_header   X-Real-IP   $remote_addr;   
                proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

> 访问`project.hellowood.com` 实际会访问该 IP 下的8080端口

## 2. 不同的 URL 访问不同的服务器

- 配置 Server 节点

```
    server {
       listen       80;
        server_name  project.hellowood.com;

        access_log  logs/project.access.log;

        location / {
                proxy_pass http://127.0.0.1:8080;
                proxy_set_header   Host    $host;  
                proxy_set_header   X-Real-IP   $remote_addr;   
                proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    server {
       listen       80;
        server_name  introduce.hellowood.com;

        access_log  logs/introduce.access.log;

        location / {
                proxy_pass http://127.0.0.1:8081;
                proxy_set_header   Host    $host;  
                proxy_set_header   X-Real-IP   $remote_addr;   
                proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    server {
       listen       80;
        server_name  helloworld.com;

        access_log  logs/helloworld.access.log;

        location / {
                proxy_pass http://127.0.0.1:8082;
                proxy_set_header   Host    $host;  
                proxy_set_header   X-Real-IP   $remote_addr;   
                proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```

> 访问`project.hellowood.com` 实际会访问该 IP 下的8080端口
> 访问`introduce.hellowood.com` 实际会访问该 IP 下的8081端口
> 访问`helloworld.com` 实际会访问该 IP 下的8082端口


----------------

## 3. 不同的项目路径访问不同的服务器

```
    server {  
        listen       80;  
        server_name  hellowood.com;  
        location / {  
            root   html;  
            index  index.html index.htm;  
        }  
        location /helloworld {  
            proxy_pass http://192.168.101.101:8080/helloworld;  
            proxy_set_header   Host    $host;  
            proxy_set_header   X-Real-IP   $remote_addr;   
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;  
        }  
        location /hi {  
            proxy_pass http://192.168.101.101:8080/hi;  
            proxy_set_header   Host    $host;  
            proxy_set_header   X-Real-IP   $remote_addr;   
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;  
        }  
        location /howareyou {  
            proxy_pass http://192.168.101.101:8088/howareyou;  
            proxy_set_header   Host    $host;  
            proxy_set_header   X-Real-IP   $remote_addr;   
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;  
        }  
        location /introduce {  
            proxy_pass http://10.2.10.11:9999/introduce;  
            proxy_set_header   Host    $host;  
            proxy_set_header   X-Real-IP   $remote_addr;   
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;  
        }  

        location /lalala {  
            proxy_pass http://www.lalala.com;  
            proxy_set_header   Host    $host;  
            proxy_set_header   X-Real-IP   $remote_addr;   
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;  
        }
        
        error_page   500 502 503 504  /50x.html;  
        location = /50x.html {  
            root   html;  
        }  
    }  
```

> 访问`hellowood.com` 实际会访问该 IP 下的 Nginx 目录下的静态页面
> 访问`hellowood.com/helloworld` 实际会访问`http://192.168.101.101:8080/helloworld` 
> 访问`hellowood.com/hi` 实际会访问`http://192.168.101.101:8080/hi` 
> 访问`hellowood.com/howareyou` 实际会访问`http://192.168.101.101:8088/howareyou` 
> 访问`hellowood.com/introduce` 实际会访问`http://10.2.10.11:9999/introduce` 
> 访问`hellowood.com/lalala` 实际会访问`http://www.lalala.com` 
