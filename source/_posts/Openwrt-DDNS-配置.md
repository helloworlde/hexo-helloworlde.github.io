---
title: Openwrt-DDNS 配置
date: 2022-08-26 11:32:08
tags:
- Esxi
- OpenWrt
- DDNS
- HomeLab
categories:
- HomeLab
---

# Openwrt DDNS 配置

DDNS(Dynamic DNS) 一般用于从外部访问家庭网络内的设备，因为家庭宽带没有固定的 IP 地址，所以通过域名访问时需要动态的更新域名的记录为当前的 IP 地址；实现的原理也比较简单，就是通过定时脚本，调用域名解析商的接口，修改域名的记录

和大部分路由器一样，OpenWrt 也支持 DDNS，通过 DDNS 的脚本执行动态更新

## 申请公网 IP

DDNS 需要能从外部访问，因此需要有一个公网IP；通常打电话给宽带运营商即可搞定，理由可以是需要访问 NAS、安装监控之类的；通常运营商会提供，获取到之后 PPPOE 重新拨号或者重启光猫路由器即可；通过访问 [https://tool.lu/ip/](https://tool.lu/ip/) 等工具，即可获取到自己的公网 IP

需要注意的是，22/80/443/8080 等常用端口被运营商限制，无法访问，因此，配置 DDNS 后只能通过域名+端口的方式访问； 如果域名是 `.dev` 这类强制要求 HTTPS 访问的域名是无法在浏览器访问的


## 配置 DDNS

### 安装 luci-app-ddns

需要安装 luci-app-ddns，用于从 OpenWrt 控制界面修改配置；通过 `opkg`命令或者 OpenWrt 控制界面的软件都可以安装

```bash
opkg update
opkg install luci-app-ddns
opkg install luci-i18n-ddns-zh-cn
```

### 安装 ddns-scripts-xxx

DDNS 的更新由脚本执行，因此需要安装对应域名服务商的更新脚本；如 godaddy 的脚本是 ddns-scripts-godaddy；官方提供的域名服务商脚本可以从 [Packagesindexnetwork---ip-addresses-and-names](https://openwrt.org/packages/index/network---ip-addresses-and-names) 查看

其他域名服务商可以在 GitHub 或恩山无线论坛中查找对应的软件

```bash
ddns-scripts-godaddy
```

### 配置 DDNS

- 添加 DDNS 配置

添加 DDNS 配置，输入 DDNS 配置名称，选择 IPV4 版本，DDNS 服务提供商选择 Godaddy

![homelab-openwrt-ddns-new-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-ddns-new-config.png)

- 配置域名信息

配置要更新的域名和对应云服务商的认证信息，保存并应用

![homelab-openwrt-ddns-basic-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-ddns-basic-config.png)


- 配置公网 IP 来源

如果是由 OpenWrt 拨号上网，IP 地址来源选择使用网络或者 WAN 接口即可；如果 OpenWrt 作为旁路由，可以通过使用 URL 或者自定义脚本进行获取；如 [http://checkip.dyndns.com](http://checkip.dyndns.com)

![homelab-openwrt-ddns-ip-address-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-ddns-ip-address-config.png)

- 修改更新时间

默认检查时间是 30 分钟，72 小时强制更新一次；可能会出现 IP 变化了，但是没有更新的情况；因此可以缩短强制更新时间，比如每隔 1小时强制更新一次

![homelab-openwrt-ddns-update-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-ddns-update-config.png)

- 启用 DDNS 配置

启用后，会自动检查 IP 信息，并执行脚本更新到 DNS 服务商，随后就可以通过域名访问了

![homelab-openwrt-ddns-enable-ddns-config-2.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-ddns-enable-ddns-config-2.png)


## 公网访问内部服务

配置 DDNS 后，通过域名访问 OpenWrt；需要注意，开启之后路由器将会完全暴露在公网，可能会被攻击或劫持；仅用于测试，不建议开启

### 配置防火墙

首先需要配置路由器的防火墙策略，允许 wan 口接受入站和转发的数据（如果 OpenWrt 是旁路由，则主路由和 OpenWrt 都需要配置防火墙；如果 OpenWrt 是主路由，配置 OpenWrt 即可）

![homelab-openwrt-ddns-enable-firewall.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-ddns-enable-firewall.png)

可以在外网通过 ping 当前的公网 IP 进行检查；通常可以使用手机开启热点，电脑连接该热点进行测试，观测 ping 返回的结果；假设当前公网 IP 地址是 `123.123.123.123`

- 如下，说明网络是通的，只需要在 OpenWrt 开启允许外网访问即可

```bash
ping 123.123.123.123
PING 123.123.123.123 (123.123.123.123): 56 data bytes
64 bytes from 123.123.123.123: icmp_seq=0 ttl=55 time=17.725 ms
64 bytes from 123.123.123.123: icmp_seq=1 ttl=55 time=16.598 ms
64 bytes from 123.123.123.123: icmp_seq=2 ttl=55 time=9.939 ms
^C
--- 123.123.123.123 ping statistics ---
8 packets transmitted, 8 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 8.355/12.869/17.725/3.600 ms
```

- 如下，说明发出的包超时了，需要检测当前的公网 IP 是否正确有效

```bash
ping 123.123.123.123
PING 123.123.123.123 (123.123.123.123): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
^C
--- 123.123.123.123 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
```

- 如下，收到回包，说明发出的包被拒绝了，大部分的路由器都不允许外网 ping，因此需要配置防火墙允许访问（辣鸡小米路由器不支持）

```
PING 123.123.123.123 (123.123.123.123): 56 data bytes
92 bytes from 123.123.123.123: Destination Port Unreachable
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
4  5  00 5400 971e   0 0000  38  01 ff90 127.0.0.1  123.123.123.123

Request timeout for icmp_seq 0
92 bytes from 123.123.123.123: Destination Port Unreachable
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
4  5  00 5400 cc8b   0 0000  38  01 ca23 127.0.0.1  123.123.123.123
```


### 添加端口转发

在 OpenWrt 的网络-防火墙-端口转发中添加配置，将路由器指定端口转发给内部服务的端口

![homelab-openwrt-ddns-add-port-forward.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-ddns-add-port-forward.png)
