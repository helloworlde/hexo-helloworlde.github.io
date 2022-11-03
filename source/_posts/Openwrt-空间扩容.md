---
title: Openwrt-空间扩容
date: 2022-07-23 11:32:08
tags:
- Esxi
- OpenWrt
- HomeLab
categories:
- HomeLab
---

# Openwrt 空间扩容

Openwrt 默认的空间只有 100M，安装一些软件后空间就被用完了，因此需要对 Openwrt 的硬盘进行扩容

有多种扩容方式，如新增一块硬盘，以USB挂载的方式扩容；或者修改虚拟机硬盘文件大小的方式扩容；此次通过修改虚拟机硬盘文件大小的方式扩容，这种方式适合新创建的虚拟机


## 修改硬盘文件大小

通过 Esxi 控制台，直接修改挂载的硬盘大小

- 在 Esxi 修改硬盘大小

![homelab-openwrt-esxi-disk-size.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-disk-size.png)


## 创建新的分区

1. 安装 `cfdisk`

需要使用 `cfdisk` 创建新的分区，所以需要安装 `cfdisk`

```bash
opkg update
opkg install cfdisk
```

2. 使用 `cfdisk`创建分区

命令行输入 `cfdisk` ，会进入到管理分区页面；可以看到，`Free Space`是新添加的硬盘大小

- 选择 `Free Space` 后选择 `New`，创建新的分区
- 输入 `Partition Size`为想要的大小
- 如`3.9G`，然后回车，此时可以看到挂载了新的分区 `/dev/sda3`

![homelab-openwrt-esxi-disk-new-partition-a-1.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-disk-new-partition-a-1.png)

选择 `/dev/sda3`，使用 Tab 选择 `Write`并输入 `yes`确认
![homelab-openwrt-esxi-disk-new-partition-a-2.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-disk-new-partition-a-2.png)

3. 格式化分区

格式化新增的 `/dev/sda3`

```bash
mkfs.ext4 /dev/sda3
```


## 挂载扩容的空间

- 安装 `block-mount`

挂载新建的分区，需要使用 `block-mount`挂载点软件，安装完成后重启 Openwrt

```bash
opkg update
opkg install block-mount
reboot
```

重启完成后访问 Web 页面的 `系统`-`挂载点`

![homelab-openwrt-esxi-disk-new-partition-a-mount-3.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-disk-new-partition-a-mount-3.png)

选择`新增`，选择刚才创建的分区`/dev/sda5`，挂载点为`作为外部 overlay 使用`，并启用

![homelab-openwrt-esxi-disk-new-partition-a-mount-1.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-disk-new-partition-a-mount-1.png)


保存并启用修改的配置，重启 Openwrt，进入挂载点，即可看到`/overlay`空间已经扩容

![homelab-openwrt-esxi-disk-new-partition-a-mount-2.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-disk-new-partition-a-mount-2.png)

## 参考文档

- [Extroot configuration](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)
