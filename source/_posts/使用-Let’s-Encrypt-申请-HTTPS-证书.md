---
title: 使用 Let’s Encrypt 申请 HTTPS 证书
date: 2022-08-03 11:32:08
tags:
- Traefik
- LetsEncrypt
- HomeLab
categories:
- HomeLab
---

# 使用 Let's Encrypt 申请 HTTPS 证书

在搭建私有服务器的过程中，需要通过外网访问，因为 .dev 域名要求使用 https，但是域名服务商只提供根域名的证书，为了使子域名也可以使用，所以通过 Let’s Encrypt 申请免费证书

Let’s Encrypt 是一家免费、开放、自动化的证书颁发机构（CA），旨在尽可能对用户友好的方式免费提供为网站启用 HTTPS（SSL/TLS）所需的数字证书

因为将域名解析迁移到腾讯云的 DNSPod 下面，所以以 DNSPod 为例，以 DNS 验证的方式，在本地机器上使用命令行申请证书

## 安装 Lego

Let’s Encrypt 有多种命令行客户端，可以使用官方提到的 [Certbot](https://certbot.eff.org/)；也可以使用 [Lego](https://go-acme.github.io/lego/)，相比 Certbot 支持的域名解析平台更多

- 安装

直接使用 Go 命令进行安装，因为本地的 Go 版本是 1.19，所以需要指定版本号使用`install`命令安装；如果是低版本的 Go，可以使用 `get`的方式安装

```bash
go install github.com/go-acme/lego/v4/cmd/lego@latest

#低版本 Go 使用 get 方式安装
go get -u github.com/go-acme/lego/v4/cmd/lego
```

## 申请证书

### 1. 创建 DNSPod Token

通过 DNS 的方式申请，Let’s Encrypt 需要向域名添加 DNS 解析以证明拥有该域名；所以需要域名解析平台的授权，通常是以 Token 或者 AK/SK 的方式进行验证

在 DNSPod 需要先申请 DNSPod Token，登录控制台后在右上角，我的账号中选择 API 密钥，然后选择 DNSPod Token 进行创建密钥

![homelab-cert-apply-dnspod.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-cert-apply-dnspod.png)


### 2. 使用 Lego 命令生成证书

通过 Lego 命令申请证书，需要指定 DNS 平台，提供用户邮箱和域名；

还需要将 DNSPod 的鉴权方式添加到变量中，key 是`DNSPOD_API_KEY`，value 是 DNSPod 生成的密钥的 ID 和 Token 的组合，是用逗号分割；如 ID 是 123，Token是 abc，那么 `DNSPOD_API_KEY` 就是 `123,abc`

指定 `DNSPOD_HTTP_TIMEOUT` 用于控制执行 HTTP 请求的超时时间，避免因为网络超时导致申请失败

```bash
DNSPOD_API_KEY=123456,abcdefghijklmnopqrstuvwxyz1234567890 DNSPOD_HTTP_TIMEOUT="300" \ lego --email helloworld@mail.com --dns dnspod --domains helloworld.dev run
```

输出以下内容：

```bash
2022/08/03 17:36:13 [INFO] [helloworld.dev] acme: Obtaining bundled SAN certificate
2022/08/03 17:36:16 [INFO] [helloworld.dev] AuthURL: https://acme-v02.api.letsencrypt.org/acme/authz-v3/123456
2022/08/03 17:36:16 [INFO] [helloworld.dev] acme: Could not find solver for: tls-alpn-01
2022/08/03 17:36:16 [INFO] [helloworld.dev] acme: Could not find solver for: http-01
2022/08/03 17:36:16 [INFO] [helloworld.dev] acme: use dns-01 solver
2022/08/03 17:36:16 [INFO] [helloworld.dev] acme: Preparing to solve DNS-01
2022/08/03 17:36:16 [INFO] [helloworld.dev] acme: Trying to solve DNS-01
2022/08/03 17:36:16 [INFO] [helloworld.dev] acme: Checking DNS record propagation using [192.168.1.1:53 172.1.1.1:53]
2022/08/03 17:36:18 [INFO] Wait for propagation [timeout: 1m0s, interval: 2s]
2022/08/03 17:36:20 [INFO] [helloworld.dev] acme: Waiting for DNS record propagation.
2022/08/03 17:36:22 [INFO] [helloworld.dev] acme: Waiting for DNS record propagation.
2022/08/03 17:36:29 [INFO] [helloworld.dev] The server validated our request
2022/08/03 17:36:29 [INFO] [helloworld.dev] acme: Cleaning DNS-01 challenge
2022/08/03 17:36:30 [INFO] [helloworld.dev] acme: Validations succeeded; requesting certificates
2022/08/03 17:36:31 [INFO] [helloworld.dev] Server responded with a certificate.
```

会将申请的证书内容写到当前目录的 `.lego`文件夹下，`certificates`文件夹下的 `helloworld.dev.crt`和`helloworld.dev.key`就是需要的证书

```bash
.
├── accounts
│   └── acme-v02.api.letsencrypt.org
│       ├── helloworld@mail.com
│       │   ├── account.json
│       │   └── keys
│       │       └── helloworld@mail.com.key
└── certificates
├── helloworld.dev.crt
├── helloworld.dev.issuer.crt
├── helloworld.dev.json
└── helloworld.dev.key
```