---
title: Raspberry Pi 4 使用 Grafana 监控
date: 2020-09-20 22:30:45
tags:
    - RaspberryPi
    - Grafana
categories: 
    - RaspberryPi
    - Grafana
---

# Raspberry Pi 4 使用 Grafana 监控

## 运行 Influxdb

Influxdb 和 Prometheus 一样都是时序数据库，不同的是它能够作为一个转发代理接受来自不同程序的消息，这里使用 Telegraf 采集数据，存放到 Influxdb 中

- 启动

挂载的目的是为了将数据保存在宿主机上，方便查询历史数据

```bash
docker run --name influxdb -d \
	-p 8086:8086 \
	-v /root/workspace/docker/influxdb:/var/lib/influxdb \
	influxdb
```

## 运行 Telegraf

Telegraf 是一个用 Golang 写的基于插件驱动的数据收集Agent，可以用来收集机器的数据

- telegraf.conf

在 telegraf 的配置文件末尾追加以下内容
telegraf 的配置可以先通过`docker run telegraf`直接启动一个，然后进入容器，从 `/etc/telegraf/`下修改

```
[[inputs.net]]

[[inputs.netstat]]

[[inputs.file]]
  files = ["/sys/class/thermal/thermal_zone0/temp"]
  name_override = "cpu_temperature"
  data_format = "value"
  data_type = "integer"
```

- 启动

Telegraf 依赖于 Influxdb，所以使用同一个网络

```bash
docker run --name telegraf -d  \
	--net=container:influxdb \
	-v /var/run/docker.sock:/var/run/docker.sock \
	-v /proc:/host/proc:ro \
	-v /opt/:/opt/ \
	-v /usr/lib/:/usr/lib/ \
	-v /root/workspace/docker/telegraf/config/telegraf.conf:/etc/telegraf/telegraf.conf \
	-e HOST_PROC=/host/proc \
	telegraf
```

## 运行 Grafana 

- 运行

```bash
docker run \
     -d \
     --name=grafana \
     -p 3000:3000 \
     grafana/grafana
```

## 修改 Influxdb 配置

在 Influxdb 中添加一个新的用户，并授予访问 telegraf 数据库的权限，用于 Grafana 拉取数据

- 进入容器

```bash
docker exec -it influxdb bash 
```

- 启动 influxdb 客户端 

```bash
influx

Connected to http://localhost:8086 version 1.8.0
InfluxDB shell version: 1.8.0
```

- 添加用户并授予权限

```bash
create user admin with password '123456' with all privileges
```

## 添加监控面板

- 访问面板

访问树莓派的地址和相应的端口 http://192.168.31.5:3000，输入用户名 `admin` 和密码 `admin`，输入新的密码后进入面板主页

- 添加数据源 

访问设置 -> DataSources -> Add Data Source，输入相应的信息

![raspberrypi-metrics-grafana-datasource.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/raspberrypi-metrics-grafana-datasource.png)

- 导入监控看板 

点击侧边栏加号，import，然后输入面板的 id [10587](https://grafana.com/grafana/dashboards/10578)，然后 load

![raspberrypi-metrics-grafana-dashboard.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/raspberrypi-metrics-grafana-dashboard.png)
