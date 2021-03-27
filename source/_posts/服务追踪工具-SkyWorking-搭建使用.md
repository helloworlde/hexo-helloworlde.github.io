---
title: 服务追踪工具 SkyWorking 搭建使用
date: 2018-07-31 23:56:24
tags:
    - SkyWalking
    - Tool
    - Trace
categories: 
    - SkyWalking
    - Tool
    - Trace
---

# 服务追踪工具 SkyWorking 搭建使用

> [SkyWalking](https://github.com/apache/incubator-skywalking) 是用于对微服务，Cloud Native，容器等提供应用性能监控和分布式调用链追踪的工具

-  [Demo](http://49.4.12.44:8080/#/monitor/dashboard)

- 截图

![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo1.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo1.png)

![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo2.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo2.png)

![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo3.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo3.png)

![https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo4.png](https://hellowood.oss-cn-beijing.aliyuncs.com/blog/SkyWalkingDemo4.png)

> 环境
> - SkyWalking 5.0.0-beat2
> - Mac OS
> - ElasticSearch 5.6.10

## 安装  ElasticSearch 

- 下载解压 ElasticSearch 5.6.10

```bash
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.10.tar.gz
tar -vxf elasticsearch-5.6.10.tar.gz
```

- 修改配置文件`config/elasticsearch.yml`

```yaml
cluster.name: CollectorDBCluster
node.name: CollectorDBCluster1
network.host: 10.0.0.34
```

需要注意的是`cluster.name`最好是`CollectorDBCluster`，`network.host`最好是局域网 IP，否则可能会在使用时出现很多问题

- 启动

```bash
./bin/elasticsearch
```

## 安装 SkyWalking 

- 下载解压 SkyWalking 5.0.0-beat2

```bash
curl -L -O https://mirrors.tuna.tsinghua.edu.cn/apache/incubator/skywalking/5.0.0-beta/apache-skywalking-apm-incubating-5.0.0-beta.tar.gz

tar -vxf apache-skywalking-apm-incubating-5.0.0-beta.tar.gz
```

- 修改配置文件`config/application.yml`，修改所有的`localhost`为局域网 IP

```yaml
cluster
    host: 10.0.0.34
    port: 10800
    contextPath: /
cache:
  caffeine:
remote:
  gRPC:
    host: 10.0.0.34
    port: 11800
agent_gRPC:
  gRPC:
    host: 10.0.0.34
    port: 11800
agent_jetty:
  jetty:
    host: 10.0.0.34
    port: 12800
    contextPath: /
analysis_register:
  default:
analysis_jvm:
  default:
analysis_segment_parser:
  default:
    bufferFilePath: ../buffer/
    bufferOffsetMaxFileSize: 10M
    bufferSegmentMaxFileSize: 500M
    bufferFileCleanWhenRestart: true
ui:
  jetty:
    host: 10.0.0.34
    port: 12800
    contextPath: /
storage:
  elasticsearch:
    clusterName: CollectorDBCluster
    clusterTransportSniffer: true
    clusterNodes: localhost:9300
    indexShardsNumber: 2
    indexReplicasNumber: 0
    highPerformanceMode: true
    bulkActions: 2000
    bulkSize: 20
    flushInterval: 10
    concurrentRequests: 2 
    traceDataTTL: 90 # Unit is minute
    minuteMetricDataTTL: 90 # Unit is minute
    hourMetricDataTTL: 36 # Unit is hour
    dayMetricDataTTL: 45 # Unit is day
    monthMetricDataTTL: 18 # Unit is month
configuration:
  default:
    applicationApdexThreshold: 2000
    serviceErrorRateThreshold: 10.00
    serviceAverageResponseTimeThreshold: 2000
    instanceErrorRateThreshold: 10.00
    instanceAverageResponseTimeThreshold: 2000
    applicationErrorRateThreshold: 10.00
    applicationAverageResponseTimeThreshold: 2000
    thermodynamicResponseTimeStep: 50
    thermodynamicCountOfResponseTimeSteps: 40
    workerCacheMaxSize: 10000
```

- 修改 `webapp/webapp.yml` Collector 地址

```yaml
server:
  port: 8080

collector:
  path: /graphql
  ribbon:
    ReadTimeout: 10000
    listOfServers: 10.0.0.34:10800

security:
  user:
    admin:
      password: admin
```

- 修改 `config/agent.config`  Collector 地址

```
agent.application_code=AppService
collector.servers=10.0.0.34:10800
logging.level=INFO
```

- 启动 Collector 和 Webapp

```bash
./bin/startup.sh
```

或单独启动 Collector 和 Webapp

```bash
./bin/collectorService.sh

./bin/webappService.sh
```

## 使用

- Jar 启动时添加 VM 参数

```
-javaagent:/path/to/apache-skywalking-apm-incubating/agent/skywalking-agent.jar -Dskywalking.agent.application_code=YOUR_APP_NAME
```

- Tomcat 修改`bin/catalina.sh`首行配置

```
# Linux/Mac
CATALINA_OPTS="$CATALINA_OPTS -javaagent:/path/to/skywalking-agent/skywalking-agent.jar"; export CATALINA_OPTS

# Win 
set "CATALINA_OPTS=-javaagent:/path/to/skywalking-agent/skywalking-agent.jar"
```
