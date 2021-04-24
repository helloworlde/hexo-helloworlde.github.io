---
title: 树莓派 4b 使用 WiFi 从 SSD Headless 启动
date: 2021-04-24 21:21:27
tags:
    - Ubuntu
    - RaspberryPi
categories: 
    - Ubuntu
    - RaspberryPi
---

# 树莓派 4 使用 WiFi 从 SSD Headless 启动

树莓派已经默认支持从 SSD 启动，可以根据官方提供的工具初始化树莓派系统并启动；尝试通过安装 Ubuntu Server，不使用网线、显示器、键盘等，从 SSD 直接启动

## 依赖

- 树莓派 4 
- Mac
- SSD 

## 安装 Ubuntu Server

### 1. 安装 Raspberry Pi Imager

Raspberry Pi Imager 是官方提供的树莓派镜像写入工具，可以通过 UI 操作，选择树莓派支持的系统，并直接写入到 SSD 或者 SD 卡中

直接从 [https://www.raspberrypi.org/software/](https://www.raspberrypi.org/software/) 下载 Raspberry Pi Imager，并在 Mac 上安装

### 2. 写入镜像

选择 Ubuntu Server 21.04 64 bit 的镜像，第一次可能需要一些时间下载镜像

![RaspberryPiImagerChooseImage.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/RaspberryPiImagerChooseImage.png)

插入硬盘后选择要写入的硬盘，并点击写入

![RaspberryPiImagerChooseDisk.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/RaspberryPiImagerChooseDisk.png)

![RaspberryPiImageWriting.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/RaspberryPiImageWriting.png)

### 3. 配置

硬盘镜像写入完成后，会挂载一个名为 `system-boot`的目录，进入该目录，修改配置

#### 挂载文件

如果可以直接在 Mac 上修改文件，可以跳过这一步；如果文件是只读的，无法修改，可以将磁盘弹出，然后手动挂载或挂载到其他的机器上修改

```bash
# 查找磁盘
fdisk -l

Disk /dev/mmcblk0: 59.63 GiB, 64021856256 bytes, 125042688 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x4ec8ea53

Device         Boot  Start     End Sectors  Size Id Type
/dev/mmcblk0p1 *      2048  526335  524288  256M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      526336 6366175 5839840  2.8G 83 Linux
```
要修改的配置就在 `/dev/mmcblk0p1` 这个用于 Boot 的目录下

- 挂载

```
# 挂载
mkdir ssd
mount /dev/mmcblk0p1 ssd 
```

#### 修改配置

- network-config 

该文件是用于网络配置，可以修改该文件，添加自己的 WiFi 设置；如 WiFi 名称是 `mywifi`，密码是 `123456`，则修改配置为：

```
version: 2
ethernets:
  eth0:
    dhcp4: true
    optional: true
wifis:
  wlan0:
  dhcp4: true
  optional: true
  access-points:
    mywifi:
      password: "123456"
```

## 启动

将 SSD 连接到树莓派，通电并启动

### 1. 查找树莓派的 IP 地址

如果可以登录路由器控制台，可以在路由器的控制台的设备列表中根据名称查找相应的 IP 地址

如果无法登录路由器控制台，可以使用 arp 命令查找，然后逐个尝试登录

```bash
arp -na

? (192.168.0.1) at 60:3a:7c:7f:b6:f8 on en0 ifscope [ethernet]
? (192.168.0.104) at 40:31:3c:b3:cb:a4 on en0 ifscope [ethernet]
? (192.168.0.107) at dc:a6:32:5f:b4:3f on en0 ifscope [ethernet]
? (192.168.0.108) at d2:bc:42:9d:5f:38 on en0 ifscope [ethernet]
? (224.0.0.251) at 1:0:5e:0:0:fb on en0 ifscope permanent [ethernet]
? (239.255.255.250) at 1:0:5e:7f:ff:fa on en0 ifscope permanent [ethernet]
```


### 2. 登录

- 登录

使用默认用户名 `ubuntu`  和密码  `ubuntu`  登录，第一次登录需要修改密码

```bash
ssh ubuntu@192.168.0.107
```

- 导入 SSH

共 GitHub 导入登录的公钥到 `~/.ssh/authorized_keys`文件中，用户名即为自己包含公钥的 GitHub ID

```bash
ssh-import-id-gh helloworlde
```


## 参考文档

- [ssh-import-id](http://manpages.ubuntu.com/manpages/bionic/man1/ssh-import-id.1.html)
- [how-to-install-ubuntu-on-your-raspberry-pi](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview)
- [树莓派 4b 无网线安装 Ubuntu 并初始化](https://helloworlde.github.io/2019/12/15/%E6%A0%91%E8%8E%93%E6%B4%BE-4b-%E6%97%A0%E7%BD%91%E7%BA%BF%E5%AE%89%E8%A3%85-Ubuntu-%E5%B9%B6%E5%88%9D%E5%A7%8B%E5%8C%96/)
- [Raspberry Pi 4 使用 USB 从 SSD 启动](https://helloworlde.github.io/2020/09/20/Raspberry-Pi-4-%E4%BD%BF%E7%94%A8-USB-%E4%BB%8E-SSD-%E5%90%AF%E5%8A%A8/)