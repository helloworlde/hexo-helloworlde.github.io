---
title: HPA 使用自定义的指标自动扩缩容
date: 2020-09-20 22:36:15
tags:
    - Kubernetes
categories: 
    - Kubernetes
---

# HPA 使用自定义的指标自动扩缩容

Kubernetes 支持使用自定义的指标作为 HPA 的依据；

KEDA 是基于事件驱动的自动扩缩容组件；主要有两部分：
1. Agent: 用于触发和停用扩缩容，通过 keda-operator 实现
2. Metrics: 用于收集指标，提供给 Agent，通过 keda-operator-metrics-apiserver 实现

KEDA 适配了多个组件，支持从 Prometheus、MySQL、MQ、Redis、自定义的组件等获取指标

## 前提

环境中安装了 metrics-server

## 安装 KEDA 组件 

```bash
helm repo add kedacore https://kedacore.github.io/charts

helm repo update​

kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
```

安装完成后会看到相关的组件

```bash
kubectl get all -n keda
​
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/keda-operator-6b7f8d7b46-tb69x                     1/1     Running   0          160m
pod/keda-operator-metrics-apiserver-58657d68db-bs94r   1/1     Running   0          160m
​
NAME                                      TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)             AGE
service/keda-operator-metrics             ClusterIP   192.168.68.139    <none>        8383/TCP,8686/TCP   159m
service/keda-operator-metrics-apiserver   ClusterIP   192.168.106.114   <none>        443/TCP,80/TCP      160m
​
NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keda-operator                     1/1     1            1           160m
deployment.apps/keda-operator-metrics-apiserver   1/1     1            1           160m
​
NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/keda-operator-6b7f8d7b46                     1         1         1       160m
replicaset.apps/keda-operator-metrics-apiserver-58657d68db   1         1         1       160m
```

## 测试

1. 创建 HPA

```
kubectl apply -f demo-api-hpa.yaml
```
2. 为 demo 配置 Service，用于访问请求

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: demo
  name: demo
  namespace: develop
spec:
  ports:
    - nodePort: 30088
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: demo
  type: NodePort

```

3. 配置 KEDA 对象

```yaml
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: demo
  namespace: develop
spec:
  scaleTargetRef:
    deploymentName: demo
    namespace: develop
  pollingInterval: 15
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.local
        metricName: http_requests_per_min
        threshold: '10'
        query: sum(rate(http_server_requests_seconds_count{application="demo"}[1m]))
```

`poolingInterval`: 从 triggers 拉取的周期
`colldownPeriod`: 缩容的等待时间，从最后一次超过阈值的 tirgger 
`minReplicaCount`: 扩缩容最小的 Pod 数量
`maxReplicaCount`: 扩缩容最大的 Pod 数量
`triggers`: 触发器，定义指标
`type`: 数据来源，支持 Prometheus、MySQL、MQ、Redis、自定义扩展等
`metricName`: 定义当前指标的名称
`threshold`: 扩缩容的阈值
`query`: 从 Prometheus 查询指标的语句

详细参考 [Scaling Deployments, StatefulSets & Custom Resources](https://keda.sh/docs/2.0/concepts/scaling-deployments/)

4. 查看 HPA 当前状态

```
kubectl get hpa -n develop
​
NAME                REFERENCE             TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-demo       Deployment/demo       2/10 (avg)   1         10        1          77m
```

5. 发起高于 10 qps 的请求

```bash
ab -c 100 -n 100000 '10.48.37.36:30088/test'
```

6. 查看扩容状态

当请求 QPS 大于 10 之后，HPA 开始扩容

```bash
kubectl get hpa -n develop
​
NAME                REFERENCE             TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-demo       Deployment/demo       59/10 (avg)   1         10        1          79m
​
kubectl get hpa -n develop
​
NAME                REFERENCE             TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-demo       Deployment/demo       33750m/10 (avg)   1         10        4          79m
```

7. 缩容

当请求 QPS 持续低于 10，时间大于 cooldownPeriod 后，HPA 开始缩容

```bash
kubectl get hpa -n develop
​
NAME                REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-demo       Deployment/demo       800m/10 (avg)   1         10        10         87m
​
kubectl get hpa -n develop
​
NAME                REFERENCE             TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-demo       Deployment/demo       7/10 (avg)   1         10        1          88m
```

## 配置多个指标

使用 QPS 和统计的 CPU 使用率作为指标进行扩缩容

KEDA 2.0 支持直接从 Pod 获取 CPU 和内存作为指标，当前 2.0 版本未发布，可以参考：[Support adding HPA Standard Metrics to Scaled Object Spec](https://github.com/kedacore/keda/issues/852)

```yaml
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: demo
  namespace: develop
spec:
  scaleTargetRef:
    deploymentName: demo
    namespace: develop
  pollingInterval: 15
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.local
        metricName: http_requests_per_min
        threshold: '1000'
        query: sum(rate(http_server_requests_seconds_count{application="demo"}[1m]))
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.local
        metricName: cpu
        threshold: '50'
        query: (sum(process_cpu_usage{application="demo"} * 32) by (application) / sum(system_cpu_count{application="demo"}) by (application)  * 100)
```

## 参考文档

- https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics
- https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis
- https://keda.sh/
- https://keda.sh/docs/1.5/scalers/prometheus/
- https://github.com/kedacore/keda


