---
title: Mac 客户端访问 Dropbox
date: 2018-10-13 12:54:07
tags:
    - Dropbox
    - Tool
categories: 
    - Dropbox
    - Tool
---

# Mac 客户端访问 Dropbox 

> 通过使用 ShadowSocks 的 PAC 代理模式可以访问到 [Dropbox](https://www.dropbox.com/) 的网页版，但是在 Mac 上下载客户端，打开后却提示无法连接

- 参考文章 [https://my.oschina.net/frankies/blog/367659](https://my.oschina.net/frankies/blog/367659) 设置更改Host 无效
- 尝试开启 ShadowSocks 全局代理，同样无效，但是遇到过 ShadowSocks 的全局代理不生效的情况，所以尝试手动配置代理


### 获取 ShadowSocks 代理配置
- 偏好设置 - 高级
![ShadowSocks 配置](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/Dropbox1.png)


### 设置 Dropbox 网络代理
- 设置 - 首选项 - 网络 - 代理服务器 - 更改设置
- 选择手动，代理类型为 SOCKS5 ，IP 和端口填写 ShadowSocks 的IP和端口

![Dropbox 配置](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/Dropbox2.png)
- 更新之后就可以正常通过客户端使用 Dropbox 了 