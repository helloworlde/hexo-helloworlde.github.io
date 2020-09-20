---
title: 使用 Prometheus 和 Grafana 监控 Consul
date: 2020-05-16 14:36:34
tags:
    - Prometheus
    - Grafana
    - Consul
categories: 
    - Prometheus
    - Grafana
    - Consul
---

# 使用 Prometheus 和 Grafana 监控 Consul

使用 Prometheus  和 Grafana 监控 Consul ，便于了解 Consul当前的状态，使用 Docker分别启动多个容器

## 启动 Consul

- 创建配置文件 

```bash
mkdir -p ~/docker/consul/config
vi ~/docker/consul/config/config.json
```

添加以下内容，目的是为了启用 Consul 的 Prometheus，否则会在调用相关端口时提示 `415 Unsupport Media Type` 和 `Prometheus is not enabled since its retention time is not positive`

```json
{
    "telemetry": {
        "prometheus_retention_time": "480h",
        "disable_hostname": true
    }
}
```

- 启动容器

将当前目录挂载到容器的config目录下

```bash
docker run \
       -d \
       --name consul \
       -v ~/docker/consul/config:/consul/config \
       -p 8500:8500 \
       consul
```

- 检查是否正常

如果正常的话会返回相应的 Prometheus 数据

```bash
curl http://127.0.0.1:8500/v1/agent/metrics\?format\=prometheus
```

## 启动 Prometheus

- 创建配置文件 

```bash
mkdir -p ~/docker/prometheus/config
vi ~/docker/prometheus/config/prometheus.yml
```

添加以下内容:

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
  - static_configs:
    - targets: []
    scheme: http
    timeout: 10s
    api_version: v1
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - localhost:9090
- job_name: consul
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: '/v1/agent/metrics'
  scheme: http
  static_configs:
  - targets:
    - host.docker.internal:8500
```

其中`targets`的值为 `host.docker.internal:8500`，目的是为了从容器中通过宿主机访问另一个容器

```bash
docker run \
	-d \
	--name prometheus \
    -p 9090:9090 \
    -v ~/docker/prometheus/config/prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus    
```

待启动后访问 [http://localhost:9090/graph](http://localhost:9090/graph)，可以看到 Prometheus的页面，选择 Status => Targets，可以看到 Consul 和 Prometheus 的状态都是UP

![consul-prometheus-targets.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/consul-prometheus-targets.png)

## 启动 Grafana 

- 启动容器

```bash
docker run \
     -d \
     --name=grafana \
     -p 3000:3000 \
     grafana/grafana
```

- 访问Grafana 

访问 [http://localhost:3000/](http://localhost:3000/)，用户名和密码都是 `admin`，输入后重新设置密码

- 创建数据源 

Dashboard 点击 `Create a datasource`，选择 Prometheus，URL 输入 `http://host.docker.internal:9090`，点击 `Save & Test`，看到访问成功

![consul-grafana-datasource.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/consul-grafana-datasource.png)


- 添加监控看板

点击左侧`+`按钮，选择 `import`，输入Dashboard ID为 `10642`，选择Prometheus为刚才添加的数据源，点击Import后即可看到监控面板

![consul-grafana-dashboard.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/consul-grafana-dashboard.png)

![consul-grafana-dashboard-data.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/consul-grafana-dashboard-data.png)

---

## 遇到的问题 

### 415 Unsupported Media Type

Consul 启动之后Prometheus的Target提示 `415 Unsupported Media Type`

- 415 Unsupported Media Type
- Prometheus is not enabled since its retention time is not positive* Closing connection 0

![consul-prometheus-target-error.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/consul-prometheus-target-error.png)

通过CURL访问数据接口，发现提示同样的问题：

```bash
curl -v localhost:8500/v1/agent/metrics\?format=prometheus
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8500 (#0)
> GET /v1/agent/metrics?format=prometheus HTTP/1.1
> Host: localhost:8500
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 415 Unsupported Media Type
< Vary: Accept-Encoding
< Date: Thu, 07 May 2020 00:09:23 GMT
< Content-Length: 66
< Content-Type: text/plain; charset=utf-8
<
* Connection #0 to host localhost left intact
Prometheus is not enabled since its retention time is not positive* Closing connection 0
```

解决方法是修改Consul的配置，将自定义的配置挂载到容器 `/consul/config`目录下

config.json

```json
{
    "telemetry": {
        "prometheus_retention_time": "480h",
        "disable_hostname": true
    }
}
```

### "INVALID" is not a valid start token

这个问题是因为数据格式不正确导致的，通过指定拉取的数据格式即可解决，在 Prometheus 的配置文件添加中`param` 参数，这样会在请求时将参数附加在链接中

```yaml
- job_name: consul
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: '/v1/agent/metrics'
  scheme: http
  param: 
    format: ["prometheus"]
  static_configs:
  - targets:
    - host.docker.internal:8500
```