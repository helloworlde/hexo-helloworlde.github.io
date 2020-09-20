---
title: 使用自定义 Grafana 面板监控 Consul
date: 2020-05-16 14:47:24
tags:
    - Prometheus
    - Grafana
    - Consul
categories: 
    - Prometheus
    - Grafana
    - Consul
---

# 使用自定义 Grafana 面板监控 Consul

使用 Prometheus和 Grafana监控 Consul，[Dashboard](https://grafana.com/grafana/dashboards?orderBy=name&direction=asc) 中的基本都是Consul 自身的状态，除此之外，还需要一些业务相关的监控，比如当前注册的服务数量，健康和不健康的服务数量，拉取服务请求响应时间等数据

## 使用已有的 Dashboard

如使用 [consul server](https://grafana.com/grafana/dashboards/10890) 这个面板，这个面板数据非常齐全，但是在 Prometheus 中添加了任务之后，发现很多数据都没有，如集群中 server的数量 `consul_serf_lan_members` 这个数据，从 Consul 的 Metrics 中 [http://localhost:8500/v1/agent/metrics?format=prometheus](http://localhost:8500/v1/agent/metrics?format=prometheus)拉取也没有相关的数据，是因为Consul并没有提供相应的数据检测

针对这种问题，可以使用 [consul_exporter](https://github.com/prometheus/consul_exporter) 这个项目，该项目会通过 Consul 的API 拉取相应的数据，在整理后通过自己的接口提供相应的统计数据

- 通过 Docker 启动

```bash
docker run --name exporter -d -p 9107:9107 prom/consul-exporter --consul.server=host.docker.internal:8500
```

- 检查数据

```bash
curl localhost:9107/metrics
```

会返回相应的监控数据，这样就可以将 Consul中未提供的数据添加到 Prometheus中了

## 自定义监控数据

如果数据仍然不满足，可以基于[consul_exporter](https://github.com/prometheus/consul_exporter) 这个项目进行扩展，添加自定义的统计数据；如现在需要统计集群的响应时间，可以通过统计请求consul的耗时来实现：

1. 添加自定义的统计项

在常量中添加一个新的统计项

```go
    responseTime = prometheus.NewDesc(
        prometheus.BuildFQName(namespace, "", "response_time"),
        "Time spend for a request ",
        []string{"node", "server_ip"}, nil,
    )
```

2. 实现统计方法 

```go
func (e *Exporter) collectResponseTime(ch chan<- prometheus.Metric) bool {
    start := time.Now().Nanosecond()
    serverIp, err := e.client.Status().Leader()
    if err != nil {
        _ = level.Error(e.logger).Log("msg", "Failed to query leader data", "err", err)
        return false
    }
    costTime := time.Now().Nanosecond() - start
    
    ch <- prometheus.MustNewConstMetric(responseTime, prometheus.GaugeValue, float64(costTime), "leader", serverIp)
    
    return true
}
```

3. 将统计项添加到 `Collect` 和 `Describe`中

```go
func (e *Exporter) Describe(ch chan<- *prometheus.Desc) {
    ch <- responseTime
}

func (e *Exporter) Collect(ch chan<- prometheus.Metric) {
    ok = e.collectResponseTime(ch) && ok
}
```

这样，就会在启动后获取相应的数据，之后在 Prometheus 和 Grafana 中可以看到相应的数据


## 自定义 Dashboard 

自定义的  Dashboard 是通过展示 [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/) 查询的结果来实现的

如在应用中有错误请求的统计，是通过累加错误的请求次数实现的，如统计值 `consul_response_time`

- 原始数据：

```bash
# HELP consul_response_time Time spend for a request
# TYPE consul_response_time gauge
consul_response_time{node="leader",server_ip="172.19.0.2:8300"} 2.238e+06
```

- 现在要统计所有的错误请求次数，可以在 Prometheus 的查询面板中查询：

```sql
consul_response_time
```
![grafana-custom-dashboard-cosnul-reponse-time-prometheus.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-custom-dashboard-cosnul-reponse-time-prometheus.png)

这样，就可以得到相应的错误数据，接下来只需要在Grafana中展示就可以

- 添加看板

添加一个 Dashboard，并添加一个 Panel，在 Panel 的 Metrics 中添加刚才的查询语句

![grafana-custom-dashboard-cosnul-reponse-time-grafana.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-custom-dashboard-cosnul-reponse-time-grafana.png)

执行查询后，会看到有图表生成，变量的名称通过 Legend 字段指定，如这里是 `{instance="host.docker.internal:9107", job="consul-exporter", node="leader", server_ip="172.19.0.2:8300"}`，需要显示IP，即 `server_ip` 的值，可以设置 Legend，这样会显示正确的名称

其他的显示单位，显示效果等以及面板的名称可以通过旁边的设置选项进行配置

## 监控服务信息

可以根据 Consul 和 consul_exporter 对服务状态进行监控，只需要根据不同的数据进行聚合配置就可以实现

- 节点信息

```sql
sum(consul_health_node_status)
```

- 健康节点信息

```sql
sum(consul_health_node_status{status="passing"})
```

- 不健康节点信息

```sql
sum(consul_health_node_status{status!="passing"})
```

- 服务信息

```sql
count(sum(consul_health_service_status) by (service_name))
```

- 实例数量 

```sql
sum(consul_health_service_status)
```

- 健康实例数量 

```sql
sum(consul_health_service_status{status="passing"})
```

- 不健康实例数量 

```sql
sum(consul_health_service_status{status!="passing"})
```

- 响应延时

```sql
consul_response_time/1000000
```

- 服务状态 

```sql
sum(consul_health_service_status{status!="passing"}) by (service_name)

sum(consul_health_service_status) by (service_name)
```

- 服务注册信息 

```sql
sum(consul_health_service_status)

sum(consul_health_service_status{status="passing"})

sum(consul_health_service_status{status!="passing"})
```

- 节点信息

```sql
sum(consul_health_node_status)

sum(consul_health_node_status{status="passing"})

sum(consul_health_node_status{status!~"passing"})
```

最终效果 

![grafana-custom-dashboard-cosnul-panel.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-custom-dashboard-cosnul-panel.png)

- 面板的 JSON文件 

根据 [Dashboard 的JSON配置文件](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/custom-consul-grafana-dashboard.json) 导入即可快速使用这个 Dashboard
