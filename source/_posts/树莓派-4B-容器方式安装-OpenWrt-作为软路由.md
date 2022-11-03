---
title: 树莓派 4B 容器方式安装 OpenWrt 作为软路由
date: 2022-07-20 11:32:08
tags:
- OpenWrt
- RaspberryPi
- HomeLab
categories:
- HomeLab
- RaspberryPi
---

# 树莓派 4B 容器方式安装 OpenWrt 作为软路由

> 在树莓派 4B，基于 Ubuntu 22.04，使用 Docker 容器的方式运行 Openwrt 作为软路由，基于 [SuLingGG/OpenWrt-Docker](https://github.com/SuLingGG/OpenWrt-Docker) 的方案

- 什么是软路由

硬路由以特有的硬设备，包括处理器、电源供应、嵌入式软件，提供设定的路由器功能，如常用的路由器；软路由则是指利用台式机或服务器配合软件形成路由解决方案，主要靠软件的设置，达成路由器的功能；

普通路由器因为硬件性能限制，无法支持长时间处理大量流量，当家中有 NAS 等设备时，通常无法跑满带宽；通过软路由，可以让路由器只处理流量的转发，其他的功能由软路由实现

通常我们使用软路由用于多线负载、宽带叠加、为局域网内的其他设备过滤广告、自定义 DNS 等扩展功能

## 安装依赖

- 安装 [linux-modules-extra-raspi](https://ubuntu.pkgs.org/21.10/ubuntu-main-arm64/linux-modules-extra-raspi_5.13.0.1008.14_arm64.deb.html)

需要保证安装了`linux-modules-extra-raspi`，否则会导致在运行容器后出现`Error response from daemon: failed to create the macvlan port: operation not supported`错误

`linux-modules-extra-raspi` 是树莓派 Ubuntu Arm 的不常用扩展，Ubuntu 最新的包管理中默认不包含扩展；因此需要单独安装

```bash
sudo apt install linux-modules-extra-raspi
```

安装之后需要重启树莓派

```bash
sudo reboot
```

## 配置网络


- 开启网卡混杂模式

默认情况下网卡只会将发送给本机的包传递到上层服务，其他的包一律丢弃；开启混杂模式后机器的网卡能够接收所有流经过它的数据流，而无论其目的地址是否是它，一般用于网络分析和路由节点；

树莓派只有一个有线接口，地址为 `eth0`，所以在 `eth0` 接口开启混杂模式

```bash
sudo ip link set eth0 promisc on
```

执行以下命令检查结果

```bash
ifconfig eth0
```

网卡 flag 信息有 `PROMISC` 表示开启成功

```
eth0: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
inet 192.168.31.2  netmask 255.255.255.0  broadcast 192.168.31.255
inet6 2408:8207:24ac:6fc0::50c  prefixlen 128  scopeid 0x0<global>
    inet6 fe80::dea6:32ff:fe5f:b43e  prefixlen 64  scopeid 0x20<link>
    inet6 2408:8207:24ac:6fc0:dea6:32ff:fe5f:b43e  prefixlen 64  scopeid 0x0<global>
        ether dc:a6:32:5f:b4:3e  txqueuelen 1000  (Ethernet)
        RX packets 2705601  bytes 1502740361 (1.5 GB)
        RX errors 0  dropped 55  overruns 0  frame 0
        TX packets 2314782  bytes 826118897 (826.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        ```


## 配置 OpenWrt 容器

1. 创建 `macvlan`

`macvlan` 是一种网卡虚拟化技术，允许在同一个物理网卡上配置多个 MAC 地址，即多个 `interface`，每个 `interface` 可以配置自己的 IP；`macvlan`直接通过以太网的 `interface` 连接到物理网络，因此性能极好

因此，软路由需要使用 `macvlan` 配合混杂模式在容器中实现路由功能

Docker 创建 `macvlan` 时要确定所在的网段，可以在路由器后台进行确认；如小米路由器常用的是 `192.168.31.0/24`网段；在创建网络时需要保证子网网段`subnet`和网关地址`gateway`参数与当前网络一致

```bash
docker network create -d macvlan --subnet=192.168.31.0/24 --gateway=192.168.31.1 -o parent=eth0 macnet
```

2. 创建容器

创建容器时需要指定网络为刚才创建的 `macnet`

```bash
docker run --restart always --name openwrt -d --network macnet --privileged   sulinggg/openwrt:rpi4 /sbin/init
```

3. 修改容器网络配置

容器成功运行后还无法直接使用，需要进入到容器中修改 OpenWrt 的网络配置

```bash
docker exec -it openwrt bash
```

修改网络

```bash
vi /etc/config/network
```

-  `ipaddr` 为 OpenWrt 分配 IP 地址，因为 OpenWrt 使用了独立的 Mac 地址，所以这个 IP 地址不能和树莓派的相同，否则会无法访问；如树莓派的 IP 地址为 `192.168.31.2`，可以将 OpenWrt 的 IP 改为 `192.168.31.4`
- `gateway` 为当前局域网的网关地址，即 `192.168.31.1`
- `dns`  需要指定 DNS，否则无法将域名解析为 IP，建议使用路由器的地址，或者使用 `114.114.114.114`等常用 DNS 服务器

需要注意的是，使用到的 IP 都应该是固定的，可以在路由器中配置 DHCP 进行 Mac 地址绑定

```bash
config interface 'lan'
option type 'bridge'
option ifname 'eth0'
option proto 'static'
option netmask '255.255.255.0'
option ip6assign '60'
option ipaddr '192.168.31.4'
option gateway '192.168.31.1'
option dns '114.114.114.114'
```

修改完成后重启容器的网络

```bash
/etc/init.d/network restart
```
重启完成后访问 [http://192.168.31.4](http://192.168.31.4)，使用账户 `root` 和密码 `password` 即可进入管理页面

4. OpenWrt 配置

登录后在 `网络-接口`中修改 `LAN` 接口，在底层基本设置中选择`忽略此接口`，使用默认路由器做 DHCP 服务器
![openwrt-config-dhcp.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/openwrt-config-dhcp.png)


## 使用 OpenWrt

有两种使用方式：

一种是将 OpenWrt 作为路由器的网关，这样所有使用路由器的设备的流量都会被 OpenWrt 处理，这种方式要求路由器支持指定网关(辣鸡小米路由器没有 root 不支持指定，且官方已不支持 root)，并且 OpenWrt 足够稳定，否则会影响使用；


另一种方式是通过在设备端指定 OpenWrt 作为网关，只有这些设备的流量会被软路由处理，这种方式需要设备端支持（大部分 IoT 设备无法配置）
![openwrt-usage-mac.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/openwrt-usage-mac.png)


## 配置重启后网卡默认使用混杂模式

网卡的混杂模式会在系统重启后失效，需要在系统启动后自定配置混杂模式，参考 [How to enable Promiscuous Mode permanently on a NIC managed by NetworkManager?](https://askubuntu.com/questions/1355974/how-to-enable-promiscuous-mode-permanently-on-a-nic-managed-by-networkmanager)，通过注册一个单独的服务启用网卡混杂模式

1. 添加 `oneshot`服务，在网络启动之后执行，配置 `eth0` 网卡为混杂模式

```bash
sudo bash -c 'cat > /etc/systemd/system/eth0-promisc.service' <<'EOS'
[Unit]
Description=Makes interfaces run in promiscuous mode at boot
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip link set dev eth0 promisc on
TimeoutStartSec=0
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOS
```

2. 启用注册的服务

```bash
sudo systemctl enable eth0-promisc
```

这样，在系统重启后依然可以正常使用 OpenWrt

## 修复宿主机与 OpenWrt 无法通信问题

Docker 的 `macvlan` 为了安全禁止宿主机与容器通信，因此宿主机和 OpenWrt 无法互相访问，导致宿主机上的一些监控无法从 OpenWrt 拉取数据；可以通过在宿主机上再创建一个 `macvlan` 接口，修改路由规则解决该问题

### 配置 macvlan 接口

- 宿主机创建 `macvlan` 接口

在宿主机上添加了一个新的 `macvlan` 桥接 `eth0` 的接口，名称为 `openwrt-bridge`

```bash
ip link add openwrt-bridge link eth0 type macvlan mode bridge
```

- 为 `openwrt-bridge` 接口配置 IP 并启用

为 `openwrt-bridge` 接口配置不冲突的 IP地址 `192.168.31.6`，并启用该接口

```bash
ip addr add 192.168.31.6 dev openwrt-bridge
ip link set openwrt-bridge up
```

- 配置路由规则，添加 OpenWrt

将 OpenWrt 的 IP 地址添加到 `openwrt-bridge` 接口的路由规则中，即可实现宿主机与 OpenWrt 容器的互相通信

```bash
ip route add 192.168.31.4 dev openwrt-bridge
```
### 配置重启后自动使用 macvlan 接口

同样通过添加一个 `oneshot`服务的方式配置

1. 添加 `oneshot`服务，在网络启动之后执行，添加 `macvlan` 网卡

```bash
sudo bash -c 'cat > /etc/systemd/system/openwrt-bridge.service' <<'EOS'
[Unit]
Description=Add network interface for openwrt at boot
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip link add openwrt-bridge link eth0 type macvlan mode bridge
ExecStart=/usr/sbin/ip addr add 192.168.31.6 dev openwrt-bridge
ExecStart=/usr/sbin/ip link set openwrt-bridge up
ExecStart=/usr/sbin/ip route add 192.168.31.4 dev openwrt-bridge
ExecStop=/usr/sbin/ip link delete openwrt-bridge
TimeoutStartSec=0
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOS
```

2. 启用注册的服务

```bash
sudo systemctl enable openwrt-bridge
```


## 参考文档

- [https://switch-router.gitee.io/blog/macvlan/](%E7%90%86%E8%A7%A3%20macvlan)
- [在Docker 中运行 OpenWrt 旁路网关](https://mlapp.cn/376.html)
- [USING DOCKER MACVLAN NETWORKS](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/)
- [What is the difference between systemd's 'oneshot' and 'simple' service types?](https://stackoverflow.com/questions/39032100/what-is-the-difference-between-systemds-oneshot-and-simple-service-types)
- [Simple vs Oneshot - Choosing a systemd Service Type](https://trstringer.com/simple-vs-oneshot-systemd-service/)