---
title: Ubuntu 搭建 ShadowSocks 服务
date: 2018-10-21 22:39:52
tags:
    - ShadowSocks
    - Tool
    - Ubuntu
categories: 
    - ShadowSocks
    - Tool
    - Ubuntu
---


# Ubuntu 搭建 ShadowSocks 服务

> 在 Ubuntu 服务器上通过脚本安装 ShadowSocks 服务

> 来自 [https://teddysun.com/342.html](https://teddysun.com/342.html)

## 安装 

- 下载安装脚本 

```
wget --no-check-certificate  https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
```

- 改变脚本执行权限 

```
chmod +x shadowsocks.sh
```

- 执行 

```
./shadowsocks.sh 2>&1 | tee shadowsocks.log
```

- 添加配置 
在执行过程中会提示设置端口，密码，加密算法等配置，根据需要自己选择即可，端口默认`8899`，加密算法可选`aes-256-cfb`，适合大多数设备使用

```
Congratulations, Shadowsocks-python server install completed!
Your Server IP        :  127.0.0.1
Your Server Port      :  8899
Your Password         :  password
Your Encryption Method:  aes-256-cfb

Welcome to visit:https://teddysun.com/342.html
Enjoy it!
```
安装完成，在客户端设备添加配置即可

- 后续修改设置可以在 `/etc/shadowsocks.json`中修改

- 多账户多端口配置，修改配置文件 `/etc/shadowsocks.json`

```
{
    "server":"0.0.0.0",
    "local_address":"127.0.0.1",
    "local_port":1080,
    "port_password":{
         "8989":"password0",
         "9001":"password1",
         "9002":"password2",
         "9003":"password3",
         "9004":"password4"
    },
    "timeout":300,
    "method":"your_encryption_method",
    "fast_open": false
}
```

## 卸载 

```
./shadowsocks.sh uninstall
```

