---
title: MacOS Monterey 制作 Windows 启动盘
date: 2022-06-30 11:37:22
tags:
- MacOS
- Windows
- ISO
- HomeLab
categories:
- HomeLab
---

# MacOS Monterey 制作 Windows 启动盘

> 需要在新的电脑安装 Windows 系统，但是手里只有 Mac，所以需要通过 Mac 制作 Windows 启动盘


> 搜到的一些方法，如启动转换助理，或者 [balenaEtcher](https://www.balena.io/etcher/) 等；但是启动转换助理在 Monterey 上不支持写入到外部硬盘；balenaEtcher 提示无法制作 Windows 镜像；又不想只为了制作启动盘单独下一个软件，所以最终通过命令行制作

## 下载并挂载 Windows ISO 镜像

1. 在微软官方网站下载 [Windows 镜像](https://www.microsoft.com/software-download/windows11)
2. 挂载 Windows 镜像到 Mac

使用 `hdiutil` 挂载 iso 镜像文件

```bash
hdiutil mount ~/Downloads/Win10_21H2_Chinese\(Simplified\)_x64.iso
```

返回以下结果， 挂载的路径`/Volumes/CCCOMA_X64FRE_ZH-CN_DV9`在复制文件时还需要用到

```bash
/dev/disk3  /Volumes/CCCOMA_X64FRE_ZH-CN_DV9
```

### 格式化 U 盘

1. 查找 U 盘路径
使用 `diskutil`查看所有挂载的硬盘，可以通过名称及容量查找；如果已经执行上面的命令挂载了 ISO 镜像，且没有其他硬盘，那么路径一般为 `/dev/disk3`

```bash
diskutil list | grep external
```

返回的 `dev/disk3` 即为挂载的路径：

```bash
/dev/disk3 (external, physical)
```

2. 格式化 U 盘

通过 `diskutil` 命令，将挂载到 `/dev/disk3` 路径的 U 盘格式化，格式为 MBR，并命名为 `WINDOWS`

```bash
diskutil eraseDisk MS-DOS "WINDOWS" MBR /dev/disk3
```

### 写入镜像

这里需要注意，因为格式化使用的是 FAT32，不支持写入超过 4G 的文件，所以需要通过 `wimlib` 拆分后写入，需要先安装 `wimlib` 工具

- 安装

```bash
brew install wimlib
```

- 写入

通过 `rsync`命令将镜像中除了 `install.wim` 之外的文件都复制到 U盘中，`install.wim`文件大小为 4.5G，超过了4G 限制，如果直接复制会失败，需要通过 `wimlib`拆分后写入

```
rsync -vha --exclude=sources/install.wim /Volumes/CCCOMA_X64FRE_ZH-CN_DV9/* /Volumes/WINDOWS
```

- 写入超出限制的文件

通过 `wimlib` 将超出限制的文件拆分复制到 U盘中，其中 `3800`是拆分的大小，单位是 `MB`

```bash
wimlib-imagex split /Volumes/CCCOMA_X64FRE_ZH-CN_DV9/sources/install.wim /Volumes/WINDOWS/sources/install.swm 3800
```

写入完成后即可在新机器上安装 Windows 了