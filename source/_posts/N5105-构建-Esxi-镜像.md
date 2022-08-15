---
title: N5105 构建 Esxi 镜像
date: 2022-08-11 10:53:05
tags:
    - Esxi
categories: 
    - HomeLab
---

# N5105 构建 Esxi 镜像

N5105 使用的是 Intel i225V 网卡，但是VMWare 官方的 Esxi 镜像里并没有该网卡的驱动，安装时会因为没有网卡导致安装失败；另外，因为买了一个国产的光威 NVME 硬盘(千万别买！)，也没有相应的驱动，安装时提示没有硬盘

经过一番搜索，发现 [VMware ESXi 7.0 U3e SLIC 2.6 & Unlocker 集成 Intel NUC 网卡、USB 网卡和 NVMe 驱动 (2022.07 更新)](https://sysin.org/blog/vmware-esxi-7-u3e-nuc-usb-nvme/#%E4%B8%8B%E8%BD%BD%E5%9C%B0%E5%9D%80-313) 里面有一个添加了驱动的镜像文件，但是分享链接是百度网盘，下载完的时候我已经单独构建好了，测试可以使用

考虑到安全问题，想通过官方的镜像添加驱动的方式自行构建镜像；需要用到 Windows 电脑和 PowerShell；手里没有 Windows 电脑的可以用虚拟机或者申请按时付费的云服务器

## 1. 下载所需的镜像和驱动
 
### 1.1 申请 Esxi 授权

Esxi 的软件下载脑洞比较清奇，需要先注册申请，填写个人隐私信息如手机号，住址，公司等，等待三五天人工审核通过后就可以下载免费版本；如果没有任何反馈，可以点击申请页下面的 [Contact us](https://www.vmware.com/support/us_support.html) 提工单给 VMWare；

可以在 [VMware vSphere Hypervisor 7.0 Download Center](https://customerconnect.vmware.com/en/downloads/details?downloadGroup=ESXI70U3D&productId=974&rPId=89003) 页面申请 7.0 版本的下载；选择 Offline Bundle 的压缩文件

![homelab-esxi-build-image-esxi-download.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-esxi-build-image-esxi-download.png)

### 1.2 下载 NVME 社区驱动

从 [Community NVMe Driver for ESXi](https://flings.vmware.com/community-nvme-driver-for-esxi) 下载 NVME 驱动
![homelab-esxi-build-image-nvme-driver.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-esxi-build-image-nvme-driver.png)

### 1.3 下载网卡社区驱动

从 [Community Networking Driver for ESXi](https://flings.vmware.com/community-networking-driver-for-esxi) 下载网卡驱动，该驱动包含 Intel I225-V 网卡的驱动

![homelab-esxi-build-image-network-driver.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-esxi-build-image-network-driver.png)


## 2. 安装 PowerCLI

PowerCLI 需要使用 PowerShell，目前只能在 Windows 平台使用，其他平台 PowerShell  都是 Core 版本，会提示 `Exception: The VMware.ImageBuilder module is not currently supported on the Core edition of PowerShell`

参考 [PowerCLI Installation Guide](https://developer.vmware.com/powercli/installation-guide)，支持离线和在线的方式进行安装；在线安装比较慢，建议使用魔法；如果没有魔法建议使用离线方式安装

### 2.1 在线安装

在 PowerShell 中运行以下命令

```powershell
Install-Module -Name VMware.PowerCLI
```

### 2.2 离线安装

需要下载 PowerCLI，在 [PowerCLI Installation Guide](https://developer.vmware.com/powercli/installation-guide) 的 Offline 下面的链接里可以[直接下载](https://developer.vmware.com/docs/15743/)

#### 2.2.1. 查找 Moudle 位置

首先需要查找 PowerShell 的 Module 位置，在 Windows 下通常有多个位置，建议使用  `C:\Program Files\WindowsPowerShell\Modules`；在 PowerShell 中运行以下命令

```powershell
$env:PSModulePath.split(";")
```

返回结果：

```powershell
C:\Users\admin\Documents\WindowsPowerShell\Modules
C:\Program Files\WindowsPowerShell\Modules
C:\Windows\system32\WindowsPowerShell\v1.0\Modules
C:\Program Files (x86)\Windows Kits\10\Microsoft Application Virtualization\Sequencer\AppvPkgConverter
C:\Program Files (x86)\Windows Kits\10\Microsoft Application Virtualization\Sequencer\AppvSequencer
C:\Program Files (x86)\Windows Kits\10\Microsoft Application Virtualization\
```

#### 2.2.2. 将 PowerCLI 解压到 Module 目录下

注意，要直接解压到 Module 所在的目录，不要在 Module 下单独创建目录，否则可能会提示 '在模块`VMware.ImageBuilder”中找到“Add-EsxSoftwareDepot”命令，但无法加载该模块`

```powershell
Expand-Archive 'C:\Users\admin\Downloads\Esxi\VMware-PowerCLI-12.3.0-17860403.zip'  -DestinationPath 'C:\Program Files\WindowsPowerShell\Modules'
```

#### 2.2.3. 解锁导入的文件

在 Module 所在的目录下执行解锁

```powershell
cd 'C:\Program Files\WindowsPowerShell\Modules'

Get-ChildItem * -Recurse | Unblock-File
```


#### 2.2.4. 检查导入结果

```powershell
Get-Module -Name VMware.PowerCLI -ListAvailable
```

执行后返回 PowerCLI 的信息，版本为安装的 12.7，目录为 `C:\Program Files\WindowsPowerShell\Modules`，说明安装成功

```powershell
    目录: C:\Program Files\WindowsPowerShell\Modules


ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Manifest   12.7.0.... VMware.PowerCLI
```


## 3. 将驱动添加到镜像文件中

### 3.1 将压缩文件添加到当前 PowerShell Session 中

- 添加 Esxi

```powershell
Add-EsxSoftwareDepot C:\Users\admin\Downloads\Esxi\VMware-ESXi-7.0U3d-19482537-depot.zip
```

返回结果：

```powershell
Depot Url
---------
zip:C:\Users\admin\Downloads\Esxi\VMware-ESXi-7.0U3d-19482537-depot.zip?index.xml
```

-  添加网卡驱动

```powershell
Add-EsxSoftwareDepot C:\Users\admin\Downloads\Esxi\Net-Community-Driver_1.2.7.0-1vmw.700.1.0.15843807_19480755.zip
```

返回结果：

```powershell
Depot Url
---------
zip:C:\Users\admin\Downloads\Esxi\Net-Community-Driver_1.2.7.0-1vmw.700.1.0.15843807_19480755.zip?index.xml
```

- 添加 NVME 驱动

```powershell
Add-EsxSoftwareDepot C:\Users\admin\Downloads\Esxi\nvme-community-driver_1.0.1.0-3vmw.700.1.0.15843807-component-18902434.zip
```

返回结果：

```powershell
Depot Url
---------
zip:C:\Users\admin\Downloads\Esxi\nvme-community-driver_1.0.1.0-3vmw.700.1.0.15843807-component-18902434.zip?index.xml
```

### 3.2 获取当前的镜像

```powershell
Get-EsxImageProfile
```

返回结果如下，其中 Name 在后续使用中需要用到

```powershell
Name                           Vendor          Last Modified   Acceptance Level
----                           ------          -------------   ----------------
ESXi-7.0U3d-19482537-no-tools  VMware, Inc.    2022/3/11 15... PartnerSupported
ESXi-7.0U3d-19482537-standard  VMware, Inc.    2022/3/29 0:... PartnerSupported
ESXi-7.0U3sd-19482531-no-tools VMware, Inc.    2022/3/11 13... PartnerSupported
ESXi-7.0U3sd-19482531-standard VMware, Inc.    2022/3/29 0:... PartnerSupported
```

### 3.3 复制新的镜像，并添加驱动

- 复制镜像

基于当前的 Esxi 镜像，复制新的镜像，指定名称和租户，便于后面区分

```powershell
New-EsxImageProfile -CloneProfile 'ESXi-7.0U3sd-19482531-standard'  -name 'ESXi-7.0U3sd-N5105' -vendor 'hellowood'
```

- 添加网卡驱动

使用复制镜像时使用的名称 `ESXi-7.0U3sd-N5105`，将网卡驱动 `net-community` 添加到镜像中

```powershell
Add-EsxSoftwarePackage -ImageProfile 'ESXi-7.0U3sd-N5105' -SoftwarePackage 'net-community'
```

返回结果：

```powershell
Name                           Vendor          Last Modified   Acceptance Level
----                           ------          -------------   ----------------
ESXi-7.0U3sd-N5105             hellowood       2022/8/11 22... PartnerSupported
```

- 添加 NVME 驱动

使用复制镜像时使用的名称 `ESXi-7.0U3sd-N5105`，将 NVME 驱动 `nvme
-community` 添加到镜像中

```powershell
Add-EsxSoftwarePackage -ImageProfile 'ESXi-7.0U3sd-N5105' -SoftwarePackage 'nvme-community'
```

返回结果：

```powershell
Name                           Vendor          Last Modified   Acceptance Level
----                           ------          -------------   ----------------
ESXi-7.0U3sd-N5105             hellowood       2022/8/11 22... PartnerSupported
```

### 3.4 将镜像导出为 iso 格式

执行导出后，就会生成 iso 格式的文件，用于制作启动盘；这样就可以在 N5105 上进行安装了

```powershell
Export-EsxImageProfile -ImageProfile 'ESXi-7.0U3sd-N5105' -ExportToIso -FilePath C:\Users\admin\Downloads\Esxi\ESXi-7.0U3sd-N5105.iso
```

## 参考文档

- [NUC 11构建 ESXi 7.0.3安装网卡驱动](https://blog.csdn.net/teamlet/article/details/124151910)
- [NUC 折腾笔记 - 安装 ESXi 7](https://soulteary.com/2021/06/22/nuc-notes-install-esxi7.html)