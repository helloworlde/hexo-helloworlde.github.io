---
title: OpenWrt-监控
date: 2022-09-26 11:32:08
tags:
- Netdata
- Prometheus
- OpenWrt
- HomeLab
categories:
- HomeLab
---

# OpenWrt 监控

OpenWrt 默认提供了内存，连接等信息，但是这些信息不够完善，不能全面的反馈 OpenWrt 的状态；可以通过使用第三方的软件来实现监控，常用的有 [Netdata](https://openwrt.org/packages/pkgdata/netdata) 和 [Prometheus](https://openwrt.org/packages/pkgdata/prometheus)

## 通过 Netdata 监控

Netdata 提供了可以在 OpenWrt 直接查看的 UI，可以查看包括 CPU，负载，内存，网络，硬盘，防火墙等常用信息的实时监控

- 安装

```bash
opkg update
opkg install netdata
```

安装完成之后，即可在 OpenWrt 的 19999 端口查看监控数据

![homelab-openwrt-metrics-netdata.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-metrics-netdata.png)


- 通过 Prometheus 抓取数据

Netdata 默认只能查看实时数据，如果想查询历史数据或者只关心特定指标，可以通过 Prometheus 抓取，使用 Grafana 查看；Netdata 支持 Prometheus 格式的数据抓取，路径是`/api/v1/allmetrics?format=prometheus_all_hosts`；如 [http://192.168.2.2:19999/api/v1/allmetrics?format=prometheus_all_hosts](http://192.168.2.2:19999/api/v1/allmetrics?format=prometheus_all_hosts)


Prometheus 任务配置：需要注意，要添加参数 `format=prometheus_all_hosts` 才可以抓取到 Prometheus 格式

```yaml
- job_name: openwrt-netdata
honor_timestamps: true
scrape_interval: 15s
scrape_timeout: 5s
metrics_path: /api/v1/allmetrics
scheme: http
params:
format: ["prometheus_all_hosts"]
static_configs:
- targets:
- 192.168.2.2:19999
```

## 通过 Grafana 监控

OpenWrt 提供了 Prometheus 数据导出的软件，安装软件后便可以配置 Prometheus 抓取数据

- 安装软件

安装需要的 Prometheus 数据导出软件

```bash
opkg update

opkg install prometheus-node-exporter-lua \
prometheus-node-exporter-lua-nat_traffic \
prometheus-node-exporter-lua-netstat \
prometheus-node-exporter-lua-openwrt
```

如果 OpenWrt 有无线网络，可以安装无线网络数据 的抓取

```bash
opkg install prometheus-node-exporter-lua-wifi \
prometheus-node-exporter-lua-wifi_stations
```

- 修改配置

Prometheus 的配置是 `/etc/config/prometheus-node-exporter-lua`，使用的接口是 `loopback`，只支持 OpenWrt 自己访问，通常我们都需要在外部访问，所以需要把接口改为 OpenWrt 对外的接口，即 `/etc/config/network` 中的 LAN 的 接口，该名称通常为 `lan`

```
config prometheus-node-exporter-lua 'main'
option listen_interface 'lan'
option listen_ipv6 '0'
option listen_port '9100'
```

- 重新启动

```bash
/etc/init.d/prometheus-node-exporter-lua restart
```

等待重启完成后，访问 `:9100/metrics` 即可获取到 Prometheus 数据，如[http://192.168.2.2:9100/metrics](http://192.168.2.2:9100/metrics)

- 配置 Prometheus 任务

```yaml
- job_name: openwrt-exporter
honor_timestamps: true
scrape_interval: 15s
scrape_timeout: 10s
metrics_path: /metrics
scheme: http
static_configs:
- targets:
- 192.168.2.2:9100
```

- 导入 Grafana 监控

相关的监控面板是 [https://grafana.com/grafana/dashboards/11147-openwrt/](https://grafana.com/grafana/dashboards/11147-openwrt/)，导入到 Grafana 即可查看监控数据

![homelab-openwrt-metrics-grafana.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/homelab-openwrt-metrics-grafana.png)

## 参考文档

- [How I monitor my OpenWrt router with Grafana Cloud and Prometheus](https://grafana.com/blog/2021/02/09/how-i-monitor-my-openwrt-router-with-grafana-cloud-and-prometheus/)