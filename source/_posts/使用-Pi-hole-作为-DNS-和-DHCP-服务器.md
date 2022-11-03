---
title: 使用 Pi-hole 作为 DNS 和 DHCP 服务器
date: 2022-09-05 11:32:08
tags:
- Pi-hole
- Docker
- DNS
- DHCP
- HomeLab
categories:
- HomeLab
---

@(HomeLab)[Pi-hole]

# 使用 Pi-hole 作为 DNS 和 DHCP 服务器

在使用 OpenWrt 的过程中，因为会经常修改 OpenWrt 的配置，导致 OpenWrt 出问题重新安装后没有来得及备份的配置丢失；其中以 IP 地址静态分配最多

另外，因为需要通过 DNS 做广告拦截，所以需要使用 Pi-hole 作为 DNS 服务器，但是 Pi-hole 提供的 DNS 服务都是国外的，所以为了快速解析国内的 DNS，需要使用 Smartdns 作为 Pi-hole DNS 的上游；DNS 的解析在 Smartdns 中提供

## 配置 Docker macvlan 网络

因为在同一个服务器上提供了多个服务，因此存在端口冲突问题，Pi-hole 和 Smartdns 都需要 53 端口用于提供 DNS，而且 53 端口默认被 Ubuntu Server 使用；而且局域网中的设备需要访问 DHCP 服务，因此为了避免冲突，需要使用 `macvlan` 作为 Docker 网络的驱动

`macvlan` 是一种网卡虚拟化技术，允许在同一个物理网卡上配置多个 MAC 地址，即多个 interface，每个 interface 可以配置自己的 IP

通过 `macvlan`，可以为每个 Docker 容器提供特定的 IP 地址，用于局域网内的设置直接通过容器的 IP 地址访问

- 开启网卡混杂模式

默认情况下网卡只会将发送给本机的包传递到上层服务，其他的包一律丢弃；开启混杂模式后机器的网卡能够接收所有流经过它的数据流，而无论其目的地址是否是它，因此，为了能让 Docker 容器能正常收到其他设备的请求，需要开启网卡混杂模式；需要注意 `eth0` 要和实际的网卡名称一致

```bash
sudo ip link set eth0 promisc on
```

- 创建 macvlan 网络

创建 macvlan 网络时，指定其 IP 范围和网关，与局域网一致，方便直接访问

```bash
docker network create -d macvlan --subnet=192.168.2.0/24 --gateway=192.168.2.1 -o parent=eth0 macnet
```

## 安装配置 Smartdns

以容器的方式运行 Smartdns，在配置中指定其上游 DNS

- smartdns.conf

在配置中添加了常用的 DNS 服务商作为上游 DNS ，可以通过 ping 的方式检查响应延时，删除超时的 DNS 上游

建议不要开启 IPV6，也不要使用 IPV6 地址作为 DNS 解析，开启后会导致网络非常明显的变慢

```
#https://github.com/pymumu/smartdns/blob/master/etc/smartdns/smartdns.conf
# 服务器名称
sever-name smartdns
# 监听端口号
bind-tcp [::]:53
bind [::]:53

# TCP 链接空闲超时时间
tcp-idle-time 3
# 域名结果缓存个数
cache-size 0
# 域名预先获取功能
prefetch-domain yes
# 过期缓存服务功能
serve-expired yes
# 过期缓存服务最长超时时间
serve-expired-ttl 0

# 测速模式选择
speed-check-mode tcp:80,tcp:443,ping
# 首次查询响应模式 fisrt-ping|fastest-ip|fastest-response
response-mode first-ping

# TTL 值
rr-ttl-min 60
rr-ttl-max 86400

# 禁止使用 IPV6 解析
force-AAAA-SOA yes
# 双栈选优
# dualstack-ip-selection yes

# 上游 UDP DNS
server 223.5.5.5
server 119.29.29.29
server 172.20.64.240
server 192.168.4.251
server 8.8.8.8 -blacklist-ip -check-edns

# 上游 TCP DNS
server-tcp 223.5.5.5
server-tcp 223.6.6.6
server-tcp 119.29.29.29
server-tcp 64.6.64.6
server-tcp 114.114.114.119
# 上游 TLS DNS
server-tls 1.1.1.1
server-tls 8.8.4.4
server-tls 9.9.9.9
# 上游 HTTPS DNS
server-https https://cloudflare-dns.com/dns-query
```

- 启动 Smardns 容器

在启动容器时指定 IP 地址为局域网内不冲突的固定 IP 地址；同时挂载配置文件到容器中

```bash
docker run --restart always \
--name smartdns \
-d --network macnet \
--ip 192.168.2.21 \
-v /workspace/smartdns/:/smartdns \
ghostry/smartdns
```

在启动完成后，使用 `telnet` 检查是否可以正常访问 53 端口

```bash
telnet 192.168.2.21 53

Trying ::1...
Connected to 192.168.2.20.
Escape character is '^]'.
```

## 安装配置 Pi-hole

- 启动 Pi-hole 容器

启动 Pi-hole 时同样需要指定 IP 地址

```bash
docker run --restart always \
--name pihole \
-d --network macnet \
--ip 192.168.2.20 \
pihole/pihole
```

启动后访问 Pi-hole 的管理界面 [http://192.168.2.20/admin](http://192.168.2.20/admin)，这时需要提供访问密码，可以进入到容器中进行修改

```bash
docker exec -it pihole bash
```

修改密码

```bash
sudo pihole -a -p
```

![homelab-dns-pihole-dashboard.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-dns-pihole-dashboard.png)

- 配置 Smartdns 作为上游服务器

在 Settings - DNS 中，配置上游的 DNS 服务，将 Smartdns 作为第一个上游服务服务；同时可以添加其他 DNS 服务，避免在 Smartdns 出现问题时使用其他 DNS 服务器
修改接口配置，允许所有的查询来源

![homelab-pihole-dns-upstream-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-pihole-dns-upstream-config.png)

- 配置 DHCP

在 Settings - DNS 中，启用 DHCP，指定分配的 IP 范围、路由器地址、本地域名以及 DHCP 过期时间

![homelab-pihole-dhcp-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-pihole-dhcp-config.png)


## 路由器配置 DHCP 和 DNS 服务

在路由器配置中， 选择关闭 DHCP，并指定 DNS 为 Pi-hole 的地址，重启路由器后，即可生效

- 关闭 DHCP

在网络-接口-LAN 中选择忽略此接口

![homelab-pihole-openwrt-close-dhcp-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-pihole-openwrt-close-dhcp-config.png)

- 配置 DNS

在网络-接口-LAN 配置中，将自定义的 DNS 服务器指向 Pi-hole 的地址

![homelab-pihole-openwrt-dns-config.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-pihole-openwrt-dns-config.png)



