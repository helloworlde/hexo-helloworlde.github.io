---
title: Openwrt 在 Esxi 中以虚拟机方式安装
date: 2022-07-22 11:32:08
tags:
- Esxi
- OpenWrt
- HomeLab
categories:
- HomeLab
---

# Openwrt 在 Esxi 中以虚拟机方式安装

## 下载镜像

在 [https://downloads.openwrt.org/](https://downloads.openwrt.org/) 选择需要下载的版本，因为 Esxi 使用的是 x86_64 平台，所以需要下载同样版本的镜像；如下载 [21.02.3](https://downloads.openwrt.org/releases/21.02.3/targets/x86/64/) 版本，路径为 `(root)/releases/21.02.3/targets/x86/64/`

选择下载 `generic-ext4-combined-efi.img.gz` 这个压缩文件，可以直接通过 EFI 引导

![homelab-openwrt-esxi-ima-download.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-ima-download.png)

## 镜像格式转换

将下载的镜像解压后得到 `img`格式的文件，这个格式无法直接被 Esxi 使用，所以需要通过 `QEMU` 软件将格式从 `img` 转为 `vmdk`

- 使用 homebrew 安装 QEMU

```bash
brew install qemu
```

- 将 `img`格式转为`vmdk`格式

```bash
qemu-img convert -f raw -O vmdk openwrt-21.02.3-x86-64-generic-ext4-combined-efi.img openwrt-21.02.3-x86-64-generic-ext4-combined-efi.vmdk
```

## 虚拟机配置

1. 创建虚拟机，选择 Linux 64 位

![homelab-openwrt-esxi-create-vm.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-create-vm.png)

2. 修改配置

需要删除硬盘，因为需要使用转换的 `vmdk` 格式的文件作为硬盘；内存和 CPU 可以根据机器自行配置

![homelab-openwrt-esxi-create-vm-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-create-vm-config.png)

3. 添加硬盘

在虚拟机的编辑界面，选择添加硬盘-现有硬盘，将转换后的 `vmdk`格式文件上传到相应目录；

需要注意，Openwrt 默认的硬盘容量只有 100M，安装软件可能空间不够；所以需要扩容，因此先不要选择该文件作为硬盘，需要扩容后才可以添加，否则容量无法修改

![homelab-openwrt-esxi-create-vm-disk.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-create-vm-disk.png)

4. 修改硬盘大小

需要登录到 Esxi 的机器上，或者通过控制台 shell，使用命令行修改

- 进入到上传文件所在的目录 `/vmfs/volumes/datastore1/OpenWrt/`
- 然后通过 `vmkfstools` 复制一个新的文件，不复制无法扩容
- 通过 `vmkfstools` 将容量修改 1G

```bash
cd /vmfs/volumes/datastore1/OpenWrt/
vmkfstools -i openwrt-21.02.3-x86-64-generic-ext4-combined-efi.vmdk openwrt.vmdk
vmkfstools -X 1G openwrt.vmdk
```

5. 再次添加硬盘
选择添加现有硬盘，将扩容后的 `openwrt.vmdk`作为硬盘文件，保存修改即可

至此，完成 Openwrt 虚拟机的创建，接下来启动虚拟机即可

## 网络配置

启动后，需要编辑 Openwrt 的网络配置才可以进行访问

- 修改网络配置

编辑 `/etc/config/network`文件，修改 `ipaddr` 为当前局域网网段的 IP；指定 `gateway`为当前网段的网关地址； `dns`可以是当前网关的地址，也可以是 DNS 服务器的地址（建议使用 DNS 服务器，网关可能无法提供 DNS 解析）

```bash
vi /etc/config/network
```

```bash
config interface 'lan'
option device 'br-lan'
option proto 'static'
option ipaddr '192.168.2.9'
option gateway '192.168.2.1'
option netmask '255.255.255.0'
option ip6assign '60'
list dns '223.5.5.5'
```

- 重启网络

重新启动网络

```bash
/ect/init.d/network restart
```

网络重启完成后，即可通过指定的地址 [192.168.2.9](192.168.2.9) 进行访问，默认账户名 `root`，没有密码
![homelab-openwrt-esxi-login.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-esxi-login.png)
