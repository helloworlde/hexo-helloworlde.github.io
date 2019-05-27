---
title: Docker 配置Ubuntu 下 Tomcat 和 Nginx 使用 HTTPS 访问
date: 2018-04-08 15:38:01
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

# Ubuntu Docker 配置 Tomcat 和 Nginx 使用 HTTPS 访问

## 安装 Docker
### 使用脚本自动安装 

```
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh --mirror Aliyun
```

### 更改镜像地址

- 修改或新建 `/etc/docker/daemon.json`

```
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```

### 启动 Docker 

```
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
```

## 配置 Tomcat
### 启动 Tomcat 容器

```
docker pull tomcat
docker run --name tomcat -d -p 8080:8080 tomcat
```

### 修改 Tomcat Manager 应用

- 修改 `webapps/manager/META-INF/content.xml`，允许需要的IP访问，这里运行所有的IP访问

```
<Context antiResourceLocking="false" privileged="true" >
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="^.*$" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
```

### 配置 Tomcat 用户

- 修改 `conf/tomcat-user.xml`，添加用户

```
<role rolename="admin-gui"/>
<role rolename="manager-gui"/>
<user username="tomcat" password="tomcat" roles="manager-gui,admin-gui"/>
```

-------------------

## 配置 Nginx 

### 配置目录

- 新建目录 `/home/ubuntu/hellowood/dev/nginx/conf`, `/home/ubuntu/hellowood/dev/nginx/log`, `/home/ubuntu/hellowood/dev/nginx/certs`
- 下载并解压相应的Nginx证书文件到 `/home/ubuntu/hellowood/dev/nginx/conf`

### 添加 Nginx 配置
- nginx.conf

```
server {
      listen 80;
      listen 443 ssl;
      server_name hellowood.com.cn;
      ssl_certificate /etc/nginx/certs/hellowood.com.cn_bundle.crt;
      ssl_certificate_key /etc/nginx/certs/hellowood.com.cn.key;
      location / {
        proxy_pass http://tomcat:8080;
      }
}
```

`http://tomcat:8080`: 将所有请求都转发到 `tomcat` 容器的 `8080`端口(不是映射端口)

### 启动 Nginx 容器

```
docker pull nginx 
docker run --name nginx -d -p 80:80 -p 443:443 \
  --link tomcat:tomcat \
  -v /home/ubuntu/hellowood/dev/nginx/conf:/etc/nginx/conf.d \ 
  -v /home/ubuntu/hellowood/dev/nginx/log:/var/log/nginx \ 
  -v /home/ubuntu/hellowood/dev/nginx/certs:/etc/nginx/certs nginx
```

此时，访问相应的域名：`http://hellowood.com.cn`和`https://hellowood.com.cn`会显示`Tomcat` 的首页，配置完成

