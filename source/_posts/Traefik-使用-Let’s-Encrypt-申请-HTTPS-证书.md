---
title: Traefik 使用 Let’s Encrypt 申请 HTTPS 证书
date: 2022-08-07 11:32:08
tags:
- Traefik
- LetsEncrypt
- HomeLab
categories:
- HomeLab
---

#  Traefik 使用 Let's Encrypt 申请 HTTPS 证书

在 Traefik 中，支持通过 Let's Encrypt 从 ACME [自动申请 HTTPS 证书](https://doc.traefik.io/traefik/https/acme/)

## 从 ACME 申请证书

 Traefik 申请证书基于 [Lego](https://github.com/go-acme/lego) ，所以同样支持基于 TLS、HTTP、DNS 三种申请方式

 因为要申请的域名没有部署服务，所以基于 DNS 的方式验证；在申请证书时会向域名解析中添加 `_acme-challenge`前缀的 TXT 记录用于验证

### 添加配置

- traefik.yaml

需要向 Traefik 的配置文件中添加 `certificatesResolvers` 的配置

```yaml
certificatesResolvers:
  default:
    acme:
      email: yourname@mail.com
      storage: /etc/traefik/certificates/acme.json
      dnsChallenge:
        provider: alidns
```

其中 `email`为注册 ACME 的邮箱，`storage` 是存储生成的证书内容的文件；`dnsChallenge` 指定了以 DNS 的方式验证；`provider` 指定域名解析平台，常见的平台参考 [providers](https://doc.traefik.io/traefik/https/ac[Ime/#providers)

- docker-compose.yaml

因为使用的是 DNS Provider 是阿里云，所以需要将阿里云的鉴权方式通过环境变量的方式添加到容器中

```yaml
version: '3'

services:
  reverse-proxy:
    container_name: reverse-proxy
    image: traefik:2.8
    command:
      - "--configFile=/etc/traefik/traefik.yml"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/root/workspaces/homelab/traefik/traefik.yml:/etc/traefik/traefik.yml"
    environment:
      - ALICLOUD_ACCESS_KEY=${ALICLOUD_ACCESS_KEY}
      - ALICLOUD_SECRET_KEY=${ALICLOUD_SECRET_KEY}
```

- 为服务指定域名

在服务路由规则中指定域名，这样 Traefik 就会为这个域名自动申请证书；需要开启 TLS 并且指定 `certresolver`，名称即为配置文件 `certificatesResolvers`中定义的名称，即 `defualt`

```yaml
version: '3'

services:
  whoami:
    image: "traefik/whoami"
    container_name: "whoami"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.homelab.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=default"
```

这样，就会在 Traefik 启动后主动为 `whoami.homelab.example.com`申请 HTTPS 证书，并保存到 `/etc/traefik/certificates/acme.json` 文件中：

```json
{
  "letsencrypt": {
    "Account": {
      "Email": "yourname@mail.com",
      "Registration": {
        "body": {
          "status": "valid",
          "contact": [
            "mailto:yourname@mail.com"
          ]
        },
        "uri": "https://acme-v02.api.letsencrypt.org/acme/acct/1234"
      },
      "PrivateKey": "xxx",
      "KeyType": "4096"
    },
    "Certificates": [
      {
        "domain": {
          "main": "whoami.homelab.example.com"
        },
        "certificate": "xxx",
        "key": "xxx",
        "Store": "default"
      }
    ]
  }
}
```


## 泛域名证书申请

在使用过程中，通常会有多个域名，需要频繁申请证书，当服务重启后同时更新可能会被 Let's Encrypt 限流导致申请失败，因此，Let's Encrypt 支持为泛域名申请证书；这个证书可以用于所有的子域名；

如要转发的 Host 都是 `xxx.homelab.example.com`，那么为 `*.homelab.example.com`申请证书即可

- traefik.yaml

同样需要先指定 `certificatesResolvers`；然后在 `entryPoints.websecure.http.tls` 中指定要申请的域名；这样 Traefik 启动后就会自动为 `homelab.example.com` 和 `*.homelab.example.com` 申请 HTTPS 证书，之后访问 `xxx.homelab.example.com` 都可以用这个证书

```yaml
certificatesResolvers:
  default:
    acme:
      email: yourname@mail.com
      storage: /etc/traefik/certificates/acme.json
      dnsChallenge:
        provider: alidns

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
    http:
      tls:
        certResolver: default
        domains:
        - main: "homelab.example.com"
          sans:
            - "homelab.example.com"
            - "*.homelab.example.com"
```

申请后得到的证书内容为：

```json
{
  "default": {
    "Account": {
      "Email": "yourname@mail.com",
      "Registration": {
        "body": {
          "status": "valid",
          "contact": [
            "mailto:yourname@mail.com"
          ]
        },
        "uri": "https://acme-v02.api.letsencrypt.org/acme/acct/1234"
      },
      "PrivateKey": "xxx",
      "KeyType": "4096"
    },
    "Certificates": [
      {
        "domain": {
          "main": "homelab.example.com",
          "sans": [
            "homelab.example.com",
            "*.homelab.example.com"
          ]
        },
        "certificate": "xxx",
        "key": "xxx",
        "Store": "default"
      }
    ]
  }
}
```

