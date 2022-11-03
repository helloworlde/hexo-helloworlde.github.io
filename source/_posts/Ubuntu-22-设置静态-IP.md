---
title: Ubuntu 22 设置静态 IP
date: 2022-9-5 11:20:19
tags:
- Ubuntu
- HomeLab
categories:
- HomeLab
---

# Ubuntu 22 设置静态 IP

在虚拟机中启动了 Ubuntu Server 作为日常使用的服务器，同时将 DHCP 和 DNS 相关的服务也运行在这个 Ubuntu Server上；


因为在 DHCP 服务中使用IP 和 Mac 绑定的方式分配 IP，因此 Ubuntu Server 是以 DHCP 方式获取 IP地址；但是在一次意外重启后，无法访问 Ubuntu Server，查看网络发现是因为网卡没有分配到 IP，这是 Ubuntu Server 依赖 DHCP 服务分配 IP，但是 DHCP 服务因为宿主机没有网络所以无法访问，造成死循环；因此通过为 Ubuntu Server 设置静态 IP 的方式，避免重启后再次出现这样的问题

## 配置静态 IP

### 1 查找网卡

通常情况下，有线网卡名称通常为 `eth0`，无线网卡名称通常为 `wlan0`；

通过 `ifconfig` 命令查看网卡信息，返回的 `ens160` 就是有线网卡

```bash
ifconfig

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
ether 02:42:e2:0c:77:47  txqueuelen 0  (Ethernet)
RX packets 0  bytes 0 (0.0 B)
RX errors 0  dropped 0  overruns 0  frame 0
TX packets 0  bytes 0 (0.0 B)
TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens160: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
ether 00:0c:29:df:81:93  txqueuelen 1000  (Ethernet)
RX packets 73618  bytes 24761397 (24.7 MB)
RX errors 0  dropped 0  overruns 0  frame 0
TX packets 27757  bytes 7800899 (7.8 MB)
TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
inet 127.0.0.1  netmask 255.0.0.0
inet6 ::1  prefixlen 128  scopeid 0x10<host>
    loop  txqueuelen 1000  (Local Loopback)
    RX packets 298  bytes 48692 (48.6 KB)
    RX errors 0  dropped 0  overruns 0  frame 0
    TX packets 298  bytes 48692 (48.6 KB)
    TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ```

### 2 修改网络配置


Ubuntu 22 的配置文件位置是 `/etc/netplan/***.yaml`，通常是 `/etc/netplan/00-installer-config.yaml`，也可能是 `50-cloud-init.yaml`；

- 00-installer-config.yaml

其中 `ens160` 是网卡名称，要和 `ifconfig`获取到的一致

```yaml
network:
ethernets:
ens160:
dhcp4: true
```

修改配置，指定静态 IP 配置信息；其中 `addresses`指定静态 IP的地址及子网信息；`routes` 指向网关地址；`nameservers`为 DNS服务器地址

```yaml
network:
ethernets:
ens160:
dhcp4: no
addresses:
- 192.168.2.4/24
routes:
- to: default
via: 192.168.2.1
nameservers:
addresses: [223.5.5.5, 8.8.8.8]
version: 2
```

### 3 应用网络配置

修改配置后，通过 `netplan` 命令使配置生效

```bash
sudo netplan apply
```

生效后，再次查看网络信息，发现 IP 地址已经修改为配置的静态 IP 地址

