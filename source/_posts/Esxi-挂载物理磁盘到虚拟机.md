---
title: Esxi 挂载物理磁盘到虚拟机
date: 2022-09-12 11:32:08
tags:
- Esxi
- HomeLab
categories:
- HomeLab
---

# Esxi 挂载物理磁盘到虚拟机

在使用 NAS 时，需要将硬盘直接挂载到 NAS 服务所在的虚拟机上；Esxi 支持将整块物理磁盘作为虚拟磁盘进行挂载

## 将物理磁盘添加为虚拟磁盘

在将磁盘连接到设备上之后，需要使用 SSH 登录 Esxi 进行操作

- 开启 SSH 登录

登录管理界面，在主机 - 操作 -服务中启用 SSH；启用成功后使用用户名密码登录到该机器

![homelab-esxi-mount-disk-to-vm.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-esxi-mount-disk-to-vm.png)

- 查看磁盘名称

通常磁盘名称以 `t10.`开头，如这里需要的机械硬盘名称为 `ATA_____Hitachi_HTS545050A7E380_______________________TE85113RHUAM6R`

```bash
ls -alh /vmfs/devices/disks/

total 1976983165
drwxr-xr-x    2 root     root         512 Sep 12 16:39 .
drwxr-xr-x   16 root     root         512 Sep 12 16:39 ..
-rw-------    1 root     root      465.8G Sep 12 16:39 t10.ATA_____Hitachi_HTS545050A7E380_______________________TE85113RHUAM6R
-rw-------    1 root     root      476.9G Sep 12 16:39 t10.NVMe____J.ZAO_5_SERIES_512GB_SSD________________091A000005275A3A
-rw-------    1 root     root      100.0M Sep 12 16:39
```

- 创建挂载目录

将需要挂载的磁盘挂载到特定的路径下，如 `datastore1/hdd` 目录，首先需要创建 `hdd`这个目录

```bash
mkdir -p /vmfs/volumes/datastore1/hdd
```

- 挂载为虚拟磁盘

使用 `vmkfstools`命令，将物理磁盘作为虚拟磁盘挂载到指定路径

```bash
vmkfstools -z /vmfs/devices/disks/t10.ATA_____Hitachi_HTS545050A7E380_______________________TE85113RHUAM6R /vmfs/volumes/datastore1/hdd/hdd.vmdk
```

## 添加磁盘到虚拟机

完成创建虚拟磁盘后，关机并编辑虚拟机，选择添加现有硬盘，将创建的虚拟磁盘作为新硬盘添加，操作完成后重启虚拟机，即可看到挂载的硬盘

![homelab-esxi-mount-disk-to-nas-vm.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-esxi-mount-disk-to-nas-vm.png)
