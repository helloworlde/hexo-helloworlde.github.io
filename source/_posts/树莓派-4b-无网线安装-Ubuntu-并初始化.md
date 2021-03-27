---
title: 树莓派 4b 无网线安装 Ubuntu 并初始化
date: 2019-12-15 22:42:04
tags:
    - Ubuntu
    - RaspberryPi
categories: 
    - Ubuntu
    - RaspberryPi
---

# 树莓派 4b 无网线安装 Ubuntu 并初始化

> 必需设备：
- 树莓派 4b
- SD 卡
- HDMI 线
- 显示器
- 键盘
- 电源及数据线

## 设置镜像 

在树莓派官网的连接，找到 Ubuntu，根据指引，找到 Ubuntu 的镜像，即[https://ubuntu.com/download/raspberry-pi](https://ubuntu.com/download/raspberry-pi)
![raspberrypi-ubuntu.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/raspberrypi-ubuntu.png)

###  下载镜像

点击下载 64 位镜像，随后会开始下载[ubuntu-19.10.1-preinstalled-server-arm64+raspi3.img.xz](http://cdimage.ubuntu.com/releases/19.10.1/release/ubuntu-19.10.1-preinstalled-server-arm64+raspi3.img.xz?_ga=2.165606655.1314896456.1576331584-894154124.1576331584)这个文件

![raspberrypi-ubuntu-download.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/raspberrypi-ubuntu-download.png)

但是这个文件下载很慢，也没有国内的镜像，可以使用迅雷下载，或者直接下载上传的镜像 [ubuntu-19.10.1-preinstalled-server-arm64+raspi3.img.xz](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/ubuntu-19.10.1-preinstalled-server-arm64+raspi3.img.xz)

### 刻录镜像

#### 格式化 SD卡

刻录镜像前，要先将 SD 卡格式化，在 Mac 上，可以使用官方推荐的[SD Card Formatter](https://www.sdcard.org/downloads/formatter/), 也可以用上传到地址进行下载Mac 版：[SDCardFormatterv5_Mac.zip](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/SDCardFormatterv5_Mac.zip)

#### 刻录镜像 

刻录镜像有多种方式，不同平台操作不同，可以参考 [https://ubuntu.com/download/iot/installation-media](https://ubuntu.com/download/iot/installation-media)

在 Mac 上，可以用官方推荐的软件[balenaEtcher](https://www.balena.io/etcher)，可以从上传的位置下载[balenaEtcher-1.5.70.dmg](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/balenaEtcher-1.5.70.dmg)，也可以直接用命令行执行

- 查找 SD 卡挂载名称

```bash
diskutil list
```

```
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *500.3 GB   disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:                 Apple_APFS Container disk1         500.0 GB   disk0s2

/dev/disk1 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +500.0 GB   disk1
                                 Physical Store disk0s2
   1:                APFS Volume Macintosh HD - 数据     280.9 GB   disk1s1
   2:                APFS Volume Preboot                 82.4 MB    disk1s2
   3:                APFS Volume Recovery                528.5 MB   disk1s3
   4:                APFS Volume VM                      8.6 GB     disk1s4
   5:                APFS Volume Macintosh HD            10.8 GB    disk1s5
/dev/disk3
#:                       TYPE NAME                    SIZE       IDENTIFIER
0:     FDisk_partition_scheme                        *32.0 GB    disk3
1:                 DOS_FAT_32 SD                      32.0 GB    disk3s1
```

其中的 `/dev/disk3`就是 SD卡

 - 取消挂载

```bash
diskutil unmountDisk /dev/disk3
```

```
Unmount of all volumes on /dev/disk3 was successful
```

- 复制镜像到 SD 卡

```bash
sudo sh -c 'gunzip -c /Users/xxx/Downloads/ubuntu-19.10.1-preinstalled-server-arm64+raspi3.img.xz | sudo dd of=/dev/disk3 bs=32m'
```

这个过程可能要十几分钟，取决于 SD 卡的速度，需要耐心等待，直到提示完成

```
3719+1 records in
3719+1 records out
3899999744 bytes transferred in 642.512167 secs (6069924 bytes/sec)
```

## 启动树莓派 

### 启动

将 SD 卡插入树莓派中，接通电源，连接 HDMI 线到显示器，并插入键盘；等待启动完成后会提示输入用户，默认的用户和密码都是`ubuntu`，进入后会要求修改密码

- 修改 root 用户密码

```bash
sudo passwd
```

根据提示输入密码即可

### 设置 WiFi

默认的树莓派会使用网线，如果有可以用的网线，直接插入，更新 WiFi 配置即可，如果不能用，则需要手动配置 WiFi

#### 查找可用的接口

```bash
ip link show 
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether dc:a6:32:5f:b4:3e brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DORMANT group default qlen 1000
    link/ether dc:a6:32:5f:b4:3f brd ff:ff:ff:ff:ff:ff
```

#### 编辑配置文件 

```bash
cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bk
vi /etc/netplan/50-cloud-init.yaml
```

此时的配置文件：

```yaml
# This file is generated from information provided by
# the datasource.  Changes to it will not persist across an instance.
# To disable cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    version: 2
```

需要新加 WiFi 的配置

```yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    wifis:
        wlan0:
            optional: true
            access-points:
               "YOUR_NETWORK_SSID":
                    password: "YOUR_PASSWORD"
            dhcp4: true
    version: 2
```
将无线网的名称和密码填写上

#### 检查配置

- 检查

```bash
sudo netplan --debug try
```
如果检查结果中没有 error，则说明正常

- 生成配置 

```bash
sudo netplan --debug generate
```
如果检查结果中没有 error，则说明正常

- 应用配置

```bash
sudo netplan --debug apply
```

设置完成后，重启树莓派

```bash
reboot now
```

待重启完成后即可自动连接上 WiFi

## 配置 SSH 

### 获取 IP 

获取 IP 可以从路由器的配置界面查看到；也可以使用在树莓派上通过命令找到

- 安装 `net-tools`

```bash
apt update 
apt install net-tools
```

- 待安装完成后，查找 IP

```bash
ifconfig 
```

```
eth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether dc:a6:32:5f:b4:3e  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 102  bytes 7390 (7.3 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 102  bytes 7390 (7.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.199.5  netmask 255.255.255.0  broadcast 192.168.199.255
        inet6 fe80::dea6:32ff:fe5f:b43f  prefixlen 64  scopeid 0x20<link>
        ether dc:a6:32:5f:b4:3f  txqueuelen 1000  (Ethernet)
        RX packets 292  bytes 78773 (78.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 181  bytes 32959 (32.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

其中的 `192.168.199.5`就是树莓派的 IP

### 配置 SSH

- 开启远程登录

默认的 22 端口是关闭的，需要手动打开

```bash
vi /etc/ssh/sshd_config
```

需要将 `#Port 22`的注释去掉，然后重启 SSH

```bash
/etc/init.d/ssh restart
```

```
[ ok ] Restarting ssh (via systemctl): ssh.service.
```

- 配置登录的 key

配置key 可以在开启了 ssh 后，通过账号密码，将 key 拷贝到树莓派

```bash
scp ~/.ssh/id_rsa.pub ubuntu@192.168.199.5:/root/.ssh/authorized_keys
```

或者上传包含自己key 的文件，在树莓派上下载

```bash
curl -O path/to/my/authorized_keys
```

-----------------

### 参考内容

- [RaspberryPi Downloads](https://www.raspberrypi.org/downloads/)
- [Install Ubuntu Server on a Raspberry Pi 2, 3 or 4](https://ubuntu.com/download/raspberry-pi)
- [Create installation media for Ubuntu](https://ubuntu.com/download/iot/installation-media)
- [balenaEtcher](https://www.balena.io/etcher/)
- [How to setup the Raspberry Pi 3 onboard WiFi for Ubuntu Server 18.04 with netplan?](https://raspberrypi.stackexchange.com/questions/98598/how-to-setup-the-raspberry-pi-3-onboard-wifi-for-ubuntu-server-18-04-with-netpla)