---
title: Kubernetes 中使用 Helm 部署使用 Prometheus
date: 2019-12-29 18:57:38
tags:
    - Kubernetes
    - Helm
    - Prometheus
categories: 
    - Kubernetes
    - Helm
    - Prometheus
---

# Kubernetes 中使用 Helm 部署使用 Prometheus

> 使用 Helm 在 Kubernetes 中部署 Prometheus，并使用 Grafana 监控集群状态，Helm 版本为 Helm3

## 安装 Prometheus 和 Grafana

### 添加标准仓库 

如果没有 stable 仓库，会提示找不到 `prometheus-operator`这个应用，需要先添加stable 仓库：

```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com
```

### 安装 Prometheus

#### 使用参数指定配置

指定节点的端口用于在集群外的机器访问

```bash
helm install prometheus stable/prometheus-operator \
	--set prometheus.service.type=NodePort \
	--set prometheus.service.nodePort=30090 \
	--set grafana.service.type=NodePort \
	--set grafana.service.nodePort=30080 \
	--set grafana.adminPassword=admin
```

#### 指定配置文件安装

- 如果有需要自定义的配置，可以下载应用后修改`values.yaml`，然后指定该配置文件进行安装 

values.yaml

```yaml
prometheus:
  service:
    nodePort: 30090
    type: NodePort

grafana:
  service:
    nodePort: 30080
    type: NodePort
  adminPassword: admin
```

安装

```bash
helm install prometheus stable/prometheus-operator -f values.yaml
```

如果有更多配置项，可以通过下载 Helm 的安装包，解压后自己修改：

```bash
helm fetch stable/prometheus-operator
```

#### 手动修改 SVC

如果在安装时没有指定使用节点端口，也可以手动修改 SVC，配置为节点端口：

##### 修改 prometheus

```bash
kubectl edit svc prometheus-prometheus-oper-prometheus
```

将 `spec.type`修改为 `NodePort`，并添加`nodePort`到 `spec.ports`中

```yaml
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: web
    nodePort: 30090
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: prometheus
    prometheus: prometheus-prometheus-oper-prometheus
  sessionAffinity: None
  type: NodePort
```

##### 修改 Grafana 

```bash
kubectl edit svc prometheus-grafana
```

```yaml
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: service
    nodePort: 30080
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: grafana
    release: prometheus
  sessionAffinity: None
  type: NodePort
```

## 配置监控

Prometheus 默认监控了 Kubernetes 和 Node，可以直接访问节点进行查看，如直接访问 [http://192.168.199.2:30090/graph](http://192.168.199.2:30090/graph)

![prometheus-homepage.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/prometheus-homepage.png)

### 使用 Prometheus 查询

通过 PromQL 查询指定的指标，可以查看数据所对应的图表，以节点 15分钟的 CPU 为例：可以直接输入`node_load15`查询，也可以从下拉框中选择，然后点击 Execute 生成图表：

![prometheus-graph.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/prometheus-graph.png)


### 使用 Grafana 查询

Prometheus 查询只能查看指定的指标，如果想要查看多个指标的聚合，或者更复杂的图表，就需要使用 Grafana 来配置查询

访问 Grafana 对应的节点，如[http://192.168.199.2:30080/?orgId=1](http://192.168.199.2:30080/)，会要求输入用户名密码，默认的用户名为 `admin`，如果没有指定`grafana.adminPassword`设置密码，则密码为`prom-operator`

![grafana-homepage.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-homepage.png)

#### 配置自定义监控 

点击左侧的加号，选择 Dashboard，添加新的查询，并输入相应的 PromQL 查询即可看到相应的指标

![grafana-query.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-query.png)

之后修改名称等其他设置，保存后就可以在面板中看到该监控了

#### 导入监控面板

因为监控的配置项有很多，配置起来也很复杂，对于一些通用的监控，如节点的运行状态等，可以直接使用已有的面板作为监控

在[https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards) 中选择需要的监控，并复制面板的 id，在Grafana 首页点击加号后选择导入，输入 id 即可导入已有的面板，如已 `11074`这个面板为例：

![grafana-import.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-import.png)

![grafana-import2.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-import2.png)

![grafana-import3.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/grafana-import3.png)