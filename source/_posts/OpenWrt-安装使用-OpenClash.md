---
title: OpenWrt  安装使用 OpenClash
date: 2022-8-25 11:20:19
tags:
- OpenClash
- Clash
- HomeLab
- OpenWrt
categories:
- HomeLab
---

#  OpenWrt 安装使用 OpenClash

## Clash 使用方式对比

OpenClash 是 Clash 的 OpenWrt 客户端；Clash 有多种使用方式，如直接使用客户端，或者以容器或进程的方式运行在服务器上，客户端以代理的方式使用，或者运行在 OpenWrt 中等；因为各种因素影响，不同的使用方式有不同的适用场景：

- 运行客户端

客户端直接使用的方式最灵活，调整代理方式或策略比较方便，可以选择性的开启或关闭，会代理设备上支持代理的所有流量；适用于手机，电脑等有客户端支持等设备

- 代理方式使用

代理方式适用于没有客户端软件的场景，如命令行，为特定的软件配置代理等；这种方式不够灵活，可按需为特定应用或设备配置

- 运行在OpenWrt

运行在 OpenWrt 等主路由或旁路由中，可以透明代理整个局域网内的流量，对于一些无法直接操作的 IoT 设备非常方便；也不需要修改客户端任何配置；缺点是如果不稳定会影响局域网内的所有设备

## 安装

需要使用 Clash 配置网络，用于访问特定的资源；[OpenClash](https://github.com/vernesong/OpenClash)  是 Openwrt 的 Clash 客户端；

1. OpenClash 依赖的是 `dnsmasq-full`，所以需要移除默认的`dnsmasq`，否则会导致 OpenClash 安装失败

```bash
opkg remove dnsmasq && opkg install dnsmasq-full
```

2. 下载并安装 OpenClash

可以在 [OpenClash](https://github.com/vernesong/OpenClash) 仓库的 [Release](https://github.com/vernesong/OpenClash/releases) 页面选择对应的版本进行下载

```bash
wget https://github.com/vernesong/OpenClash/releases/download/v0.45.35-beta/luci-app-openclash_0.45.35-beta_all.ipk -O openclash.ipk
opkg update
opkg install openclash.ipk
```

3. 添加 `luci-compact` 并重启，否则会提示进入 luci 页面错误

```bash
opkg install luci luci-base luci-compat
reboot
```

待重启完成后重新登录控制台，可以在服务菜单中看到 `OpenClash`

![homelab-openwrt-openclash-init-page.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-openclash-init-page.png)

## 配置

### 配置文件订阅

进入服务-OpenClash，选择配置文件订阅，新增订阅；输入订阅的链接地址，保存并更新即可；这样以 OpenWrt 为网关的设备就可以通过 OpenClash 访问特定的资源了

![homelab-openwrt-init-openclash-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-init-openclash-config.png)

### 配置 DNS

OpenClash 默认提供 7874 端口用于 DNS 查询；启动后会劫持 Dnsmasq，只保留自己作为 Dnsmasq 的上游；

但是目前的版本里并没有将已配置的 DNS 转发作为 OpenClash 的上游（参考issue [启用 DNS 劫持后未将 Dnsmasq 中添加的 DNS 转发作为上游 DNS](https://github.com/vernesong/OpenClash/issues/2720)），这样会导致无法使用 SmartDNS 或其他上游 DNS，因此需要手动修改将 SmartDNS 作为 OpenClash 的上游服务器

![homelab-openwrt-openclash-dnsmasq-upstream.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-openclash-dnsmasq-upstream.png)

- 配置 DNS 服务器

在 服务- OpenClash-全局设置-DNS设置中，选择新增，设置自定义上游 DNS服务器为 SmartDNS
![homelab-openwrt-openclash-add-upstream.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-openclash-add-upstream.png)

- 启用自定义上游 DNS 服务器

新增完成后，在该页面选择启用 "自定义上游 DNS 服务器"，这样，就可以使用 SmartDNS 作为主 DNS服务器了；如果有其他的上游，也可以同样配置；
![homelab-openwrt-openclash-enable-dns-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-openclash-enable-dns-config.png)
配置完成后，DNS的查询流程为 客户端 -> Dnsmasq -> OpenClash -> SmartDNS ->  上游 DNS 服务器

### 配置运行模式

参考 [OpenClash-常规设置](https://github.com/vernesong/OpenClash/wiki/%E5%B8%B8%E8%A7%84%E8%AE%BE%E7%BD%AE)，主要有 Fake-IP 和 Redir-Host 两种模式；经测试，开启了 OpenClash 使用 Redir-Host 会导致部分 UDP 流量超时，如王者荣耀/英雄联盟/吃鸡等使用 UDP 的应用超时或无法连接；使用  Fake-IP TUN 模式则可以正常使用

- Fake-IP

当客户端发起请求查询 DNS 时，会先返回一个随机的保留地址，同时查询上游 DNS 服务器，如果需要代理则发送给代理服务器查询，然后再进行连接；客户端立即向Fake-IP 发起的请求会被快速响应，节约了一次本地向DNS服务器查询的时间

运行模式有 TUN，增强和混合三种模式；区别在与 TUN 可以代理 UDP流量

这个模式会导致客户端获取到的 DNS 查询到的结果与实际不一致，`nslookup`/`dig`等的使用会受影响


- Redir-Host

当客户端发起请求时，会并发查询 DNS，等待返回结果后再尝试进行规则判定和连接，如果需要代理，会使用fallback 的 DNS 服务器再次查询；与不使用 OpenClash 相比，多了过滤，fallback 查询的时间，响应速度可能会变慢

有兼容，TUN 和混合三种模式，区别在与 TUN 可以代理 UDP流量

![homelab-openwrt-openclash-change-mode.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-openclash-change-mode.png)