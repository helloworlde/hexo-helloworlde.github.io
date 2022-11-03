---
title: Openwrt-初始化配置
date: 2022-08-08 11:44:37
tags:
- Esxi
- OpenWrt
- HomeLab
categories:
- HomeLab
---

# Openwrt-初始化配置

## 语言配置

Openwrt 默认语言为英文，如果需要安装中文，可以直接通过 `opkg` 安装；随后登录控制台即可看到语言已经变为中文；如果没有改变，可以在System-System-Language 中选择简体中文

```bash
opkg update
opkg install luci-i18n-base-zh-cn
```
![homelab-openwrt-init-config-language.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-init-config-language.png)

## 时间设置

在系统-系统-常规设置中，将时区设置为 `Asia/Honkong` ，选择保存并应用即可生效

![homelab-openwrt-init-config-time.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-init-config-time.png)

## 主题配置

Openwrt 默认的主题为 Bootstrap，菜单在上边栏，使用不习惯，可以安装 [Argon](https://github.com/jerrykuku/luci-theme-argon) 主题

可以在 GitHub [Argon](https://github.com/jerrykuku/luci-theme-argon) 仓库的 [Release](https://github.com/jerrykuku/luci-theme-argon/releases) 页面选择对应的版本进行下载

```bash
wget https://github.com/jerrykuku/luci-theme-argon/releases/download/v2.2.9.4/luci-theme-argon-master_2.2.9.4_all.ipk -O argon.ipk
opkg install argon.ipk
```

![homelab-openwrt-init-config-theme.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-init-config-theme.png)


## 设置密码

登录控制台，在系统-管理权页面，选择路由器密码；设置密码之后就可以通过账户 `root`和设置的密码进行登录

同时也可以将本地的公钥添加到 SSH 密钥中，方便登录控制台

![homelab-openwrt-init-config-password.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-init-config-password.png)
