---
title: Raspberry Pi 4 使用 USB 从 SSD 启动
date: 2020-09-20 22:29:28
tags:
    - RaspberryPi
categories: 
    - RaspberryPi
---

# Raspberry Pi 4 使用 USB 从 SSD 启动

树莓派 4 的最新固件已经支持从USB 启动，通过外接 U盘或者硬盘，能够摆脱 SD 卡的IO 速度限制，这里通过 USB 从 SSD 硬盘启动系统

## 安装 Raspberry Pi OS

- 下载 Imager 

从 [https://www.raspberrypi.org/downloads/](https://www.raspberrypi.org/downloads/) 下载相应 Imager

- 安装 Raspberry Pi OS 到 SD 卡中

选择第一个镜像

![RaspberryPiOS-install-1.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/RaspberryPiOS-install-1.png)

然后选择 SD 卡后写入

![RaspberryPiOS-install-2.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/RaspberryPiOS-install-2.png)

待写入完成后，将 SD 卡插入树莓派 4，正常启动

## 更新 EEPROM

- 查看配置

```bash
vcgencmd bootloader_version
Apr 16 2020 18:11:26
version a5e1b95f320810c69441557c5f5f0a7f2460dfb8 (release)
timestamp 1587057086
```

如果日期是 `May 15 2020` 之前的，则需要修改配置以启用新的固件

- 更新 

```bash
sudo apt update
sudo apt full-upgrade
sudo reboot now
```

等更新完成后，会安装新的 `rpi-eeprom`，更新重启后的版本是 `Jun 15 2020`

- 查看配置

```bash
vcgencmd bootloader_config

[all]
BOOT_UART=0
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0
DHCP_TIMEOUT=45000
DHCP_REQ_TIMEOUT=4000
TFTP_FILE_TIMEOUT=30000
ENABLE_SELF_UPDATE=1
DISABLE_HDMI=0
SD_BOOT_MAX_RETRIES=1
USB_MSD_BOOT_MAX_RETRIES=1
BOOT_ORDER=0xf41
```

其中的 `BOOT_ORDER`的值是`0xf41`，说明首先从`USB mass storage boot`启动，如果失败，则从`SD CARD`启动，具体的配置解释可以参考 [Pi 4 Bootloader Configuration](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711_bootloader_config.md)

## 配置硬盘

- 拷贝 SD 卡的内容到硬盘

点击左上角的树莓派图标，选择附件 -> SD Card Copier 
然后选择 From 和 To Device 为相应的 SD 卡和硬盘，点击 Start 开始复制

- 覆盖系统文件
下载 [raspberrypi/firmware](https://github.com/raspberrypi/firmware)  master 分支，解压后将 boot 目录下的所有 `dat` 和 `elf` 后缀的文件拷贝到硬盘中，替换原有的内容，这么因为镜像中的文件尚未支持 USB 启动，所以需要替换，待镜像中支持后，这个操作就可以省略了

- 从 USB 启动
关闭树莓派，拔出 SD 卡，连接硬盘后重新启动，就可以从 USB 硬盘启动系统了


## 参考文章

- [Raspberry Pi 4 boot EEPROM](https://www.raspberrypi.org/documentation/hardware/raspberrypi/booteeprom.md)