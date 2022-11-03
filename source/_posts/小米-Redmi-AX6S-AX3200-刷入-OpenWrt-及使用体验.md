---
title: 小米 Redmi AX6S(AX3200) 刷入 OpenWrt 及使用体验
date: 2022-10-09 11:20:19
tags:
- MiWiFi
- Router
- HomeLab
- OpenWrt
categories:
- HomeLab
---

# 小米 Redmi AX6S(AX3200) 刷入 OpenWrt 及使用体验

买了小米 Redmi AX3000 路由器后发现是 RA81 型号，这个型号目前无法刷入 OpenWrt；但是因为当前使用的 OpenWrt 作为旁路由的方式并不稳定，所以想更换一个可以刷入 OpenWrt 的路由器

一番搜索之后发现 Redmi AX6S 相对更稳定，且已有 OpenWrt 的正式版本固件，因此又购入了一台新的 AX6S，准备刷入 OpenWrt 作为主路由使用

AX6S有 RB01 和 RB03 两个版本，硬件完全相同；国内售卖的基本都是 RB03，可以通过刷入开发版固件的方式开启 telnet，RB01 只能拆机后通过 UART 接口写入的方式开启 telnet

## 刷入 OpenWrt

### 将固件升级为测试版

将固件升级为测试版是为了开启 telnet，方便输入 OpenWrt 的固件；可以通过 `telnet 192.168.31.1` 测试，通常是没有开启的。

```bash
telnet 192.168.31.1
Trying 192.168.31.1...
telnet: connect to address 192.168.31.1: Connection refused
telnet: Unable to connect to remote host
```


1. 下载测试版固件并升级

首先下载测试版本固件[miwifi_rb03_firmware_stable_1.2.7 ](https://github.com/YangWang92/AX6S-unlock/raw/master/miwifi_rb03_firmware_stable_1.2.7.bin)；在初始化路由器配置后，进入到路由器控制台，选择升级；
![homelab-miwifi-ax6s-openwrt-upgrade.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-miwifi-ax6s-openwrt-upgrade.png)

![homelab-miwifi-ax6s-openwrt-upgrade-ongoing.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-miwifi-ax6s-openwrt-upgrade-ongoing.png)

升级完成后路由器会重启，页面有水印提示 `Only For Test`；
![homelab-miwifi-ax6s-openwrt-upgrade-complted.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-miwifi-ax6s-openwrt-upgrade-complted.png)

此时使用 telnet 测试会发现已经可以登陆了

```bash
telnet 192.168.31.1
Trying 192.168.31.1...
Connected to xiaoqiang.
Escape character is '^]'.

XiaoQiang login:
```

2. 计算路由器登陆密码

路由器的登陆密码与 SN码有关，需要通过 SN码进行计算，可以使用 [YangWang92/AX6S-unlock](https://github.com/YangWang92/AX6S-unlock/blob/master/unlock_pwd.py) 仓库的 Python 脚本计算；SN码可以从路由器页面或者路由器背面商品信息中获取

```bash
python3 unlock.py 36418/J1VU79634
a857e591
```

使用账户 `root` 和计算得到的密码 `xxx` 进行登陆，即可登陆到路由器后台

![homelab-miwifi-ax6s-openwrt-telnet-success.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-miwifi-ax6s-openwrt-telnet-success.png)

### 刷入 OpenWrt

#### 1 下载 OpenWrt 固件

##### 直接下载到路由器

如果路由器能够上网，可以直接通过 `wget` 命令从 [Xiaomi AX3200 / Redmi AX6S](https://openwrt.org/toh/xiaomi/ax3200) 页面下载最新的安装固件 [openwrt-22.03.0-mediatek-mt7622-xiaomi_redmi-router-ax6s-squashfs-factory.bin](http://downloads.openwrt.org/releases/22.03.0/targets/mediatek/mt7622/openwrt-22.03.0-mediatek-mt7622-xiaomi_redmi-router-ax6s-squashfs-factory.bin)，为了后续操作方便，重命名为 `factory.bin`

```bash
cd /tmp

wget http://downloads.openwrt.org/releases/22.03.0/targets/mediatek/mt7622/openwrt-22.03.0-mediatek-mt7622-xiaomi_redmi-router-ax6s-squashfs-factory.bin -O factory.bin
```

##### 从本地上传

如果路由器不能上网，可以手动下载安装固件[openwrt-22.03.0-mediatek-mt7622-xiaomi_redmi-router-ax6s-squashfs-factory.bin](http://downloads.openwrt.org/releases/22.03.0/targets/mediatek/mt7622/openwrt-22.03.0-mediatek-mt7622-xiaomi_redmi-router-ax6s-squashfs-factory.bin)后上传到路由器

可以通过 [Cyberduck](https://cyberduck.io/download/) 或者本地启动一个 HTTP服务器的方式上传固件到路由器；以通过启动 Python 服务器的方式为例上传到路由器

- 启动服务器

在 OpenWrt 固件所在目录启动 Python Server； 启动后，会在本地 8000 端口提供一个 ftp 服务，可以从路由器上访问该端口获取

```bash
python -m http.server
```

- 在路由器下载固件

需要知道本地的IP地址，可以在路由器页面查看，如本地的地址是 `192.168.31.100`，然后使用 `telnet` 登陆到路由器，将 OpenWrt 估计下载到 `/tmp`目录

```bash
cd /tmp
wget http://192.168.31.100/factory.bin -O factory.bin
```


#### 2 开启 SSH 并准备刷入 OpenWrt

登陆后，执行以下命令，准备输入 OpenWrt 固件

```bash
nvram set ssh_en=1
nvram set uart_en=1
nvram set boot_wait=on
nvram set flag_boot_success=1
nvram set flag_try_sys1_failed=0
nvram set flag_try_sys2_failed=0
nvram set "boot_fw1=run boot_rd_img;bootm"
nvram commit
```

#### 3 刷入 OpenWrt

通过 `mtd` 命令将 OpenWrt 的固件写入到路由器

```bash
mtd -r write factory.bin firmware
```

![homelab-miwifi-ax6s-openwrt-execute-install.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-miwifi-ax6s-openwrt-execute-install.png)

写入完成后，路由器会自动重启；重启后的路由器的 IP 地址变为 `192.168.1.1`；此时 Wi-Fi 没有开启，需要使用网线连接到路由器，访问  `192.168.1.1` 进行配置


## 使用体验

以 Redmi AX3000 和 Redmi AX6S 日常使用体验对比

- 配置

两者硬件配置基本相同，AX6S CPU 频率更高，支持 4x4 通道 80Mhz 的 5G WiFi，AX3000 支持 2x2 160Mhz 的 5G WiFi；可以理解为 AX3000 单设备速度上限更高，AX6S 支持更多设备并发，两者理论速度上限相同

|配置| AX3000| AX6S |
|:--|:---|:----|
|CPU|IPQ5000双核1Ghz|MT7622B双核1.35Ghz|
|内存|256M|256M|
|闪存|128M|128M|
|尺寸|247x141x180mm|284x186x186mm|
|网口|千兆WANx1/LANx3|千兆WANx1/LANx3|
|无线|2x2 160Mhz,最高可达2402Mbps，4根天线|4x4 80Mhz,最高可达2402Mbps，6根天线|
|价格|239|299|

- 外观

Redmi AX3000 外观是白色，颜值在线；Redmi AX6S 使用黑色，要比 Redmi AX3000 大不少，因此显得十分笨重，但是其内部的 PCB 板周围仍有很大空间，完全可以做的更小；（AX6S太丑了！！！）

两者的网口采用了同样的设计，网口上下两面都有凹槽，使用时很容易插反（为啥要这么设计？？？）；都是一个 WAN 口，三个 LAN 口（完全不够用）

另外都有 Mesh 按钮，支持快速 Mesh 组网；但是 Reset 按钮必须得用卡针才能触发（垃圾！！！）

- 散热

因为完全镂空的设计，两个路由器的散热都非常好；高负载使用也不会烫手，相比小米路由器 4C 没有高负载也特别烫手，这两款路由器的散热提升不小

- 使用体验

Redmi AX6S 刷了 OpenWrt 后安装了 OpenClash/DDNS/SmartDNS 等几个常用的软件；内存日常占用在 80% 左右；使用 Youtube 看 8K 视频，CPU 占用大约 30%-50%，低负载时 10%以内；日常使用较稳定，因为节点偶尔抖动，所以玩游戏时 ping 可能会突然增高

Redmi AX3000 日常使用比较稳定，因为不支持刷入 OpenWrt，并且官方固件阉割了系统状态，所以无法观测 CPU 和内存等信息

速度方面，WiFi6 的电脑通过无线连接到路由器，然后通过路由器的网口连接到另一个 2.5G 网口都电脑进行测速；连接 AX3000 的无线协商速度是 2400Mbps，测速上下行大概都是 950Mbps 左右（所以千兆网口的路由器上 WiFi6 有个卵用？？？如果用于内网跑链路聚合，网口又不够用！！！）；AX6S 的协商速度是 1200Mbps，测试下行大约900Mbps，上行 850Mbps；直观感受是 AX6S速度不如AX3000，不过日常使用已经足够了

综合来看，刷入 OpenWrt 安装常用软件后，128M 的闪存和 256M 的内存不够用，使用起来捉襟见肘；另外，只有千兆的网口，导致 WiFi6 的速度只能在内网支持 WiFi6 的设备之间使用，所以这个价位的 WiFi6 路由器只能是过渡产品，可能一两年后就需要淘汰（不如小黄鱼买 Redmi AX3000 RA69 版本，千万别买 RA81 版本！！！）

![homelab-miwifi-redmi-ax6s-usage.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-miwifi-redmi-ax6s-usage.png)

## 参考文档

- [Xiaomi AX3200 / Redmi AX6S](https://openwrt.org/toh/xiaomi/ax3200)