---
title: OpenWrt 使用 SmartDNS
date: 2022-09-27 11:32:08
tags:
- SmartDNS
- DNS
- OpenWrt
- HomeLab
categories:
- HomeLab
---


# OpenWrt 使用 SmartDNS

[SmartDNS](https://pymumu.github.io/smartdns/)  是由国内用户开发的本地 DNS 服务器，从多个上游获取 DNS 结果，并将访问速度最快的地址返回给客户端；SmartDNS 可以运行在多个平台，如 Linux, OpenWrt 等

在 OpenWrt 中运行 SmartDNS，将其作为 dnsmasq 的上游或作为唯一的 DNS 服务器，用于提升 DNS 解析速度

## 安装

SmartDNS 的安装非常简单，使用 opkg 命令即可安装

```bash
opkg update
opkg install smartdns
opkg install luci-app-smartdns
opkg install luci-i18n-smartdns-zh-cn
```

## 配置 SmartDNS

安装完成后，在服务-SmartDNS 常规配置中，选择启用 SmartDNS，然后添加上游 DNS 服务器（也可以直接在命令行修改 `/etc/config/smartdns` 配置文件）

![homelab-openwrt-dns-smartdns-upstream.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-dns-smartdns-upstream.png)

这样，SmartDNS 会运行在路由器的 6053 端口上

## 配置 Dnsmasq

OpenWrt 默认的 DHCP 和 DNS 服务由 Dnsmasq 提供，所以需要配置 SmartDNS 作为 Dnsmasq 的上游 DNS 服务器

在网络-DHCP/DNS -常规设置中，添加 DNS 转发，将 SmartDNS 作为 Dnsmasq 的上游
![homelab-oepnwrt-smart-dns-as-dnsmasq-upstream.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-oepnwrt-smart-dns-as-dnsmasq-upstream.png)

## 检查配置结果

通过 `nslookup` 命令查找 `smartdns`，如果返回 `smartdns` 的解析结果，则说明配置成功

```bash
nslookup -querytype=ptr smartdns

Server:         192.168.2.1
Address:        192.168.2.1#53

Non-authoritative answer:
smartdns        name = smartdns.
```

## 将 SmartDNS 作为主 DNS 服务器

上述配置完成后，DNS的查询流程为 客户端 -> Dnsmasq -> SmartDNS ->  上游 DNS 服务器；Dnsmasq 变成了赚差价的中间商，会影响查询速度；因此，可以将 SmartDNS 作为主 DNS 服务器，不再经过 Dnsmasq

需要注意的是，如果使用了 OpenClash 等需要劫持 DNS 的服务，不能用这种方式，否则可能会导致服务无法正常使用

- 修改 Dnsmasq 的端口为非 53 端口

在网络 - DHCP/DNS 配置- 高级设置中，修改 DNS 服务器端口，改为 53 之外的其他端口，如 6653

![homelab-oepnwrt-smart-dns-masqdns-non-53-port.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-oepnwrt-smart-dns-masqdns-non-53-port.png)

- 修改 SmartDNS 的端口为 53 端口

在服务-SmartDNS-常规设置中，将 SmartDNS 的端口修改为 53，这样客户端的 DNS 查询请求都会由 SmartDNS 提供

![homelab-oepnwrt-smart-dns-use-the-53-port.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-oepnwrt-smart-dns-use-the-53-port.png)

此时 DNS 的查询流程为 客户端 -> SmartDNS ->  上游 DNS 服务器