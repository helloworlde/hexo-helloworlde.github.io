---
title: Kubenetes 部署 Dashboard
date: 2019-09-08 19:28:25
tags:
    - Kubernetes
categories: 
    - Kubernetes
---

# Kubenetes 部署 Dashboard

> [Kubenestes Dashboard](https://github.com/kubernetes/dashboard) 是提供 Kubernetes信息可视化的 Web 插件

## 部署 

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```

##  配置

### 修改为通过 NodePort 访问

```bash
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```

在 `ports`下面添加`nodePort: 32576`，将 `clusterIp`改为`NodePort`

```yaml
spec:
  clusterIP: 10.104.3.252
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32576
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
```

此时可以通过节点 IP 和端口[https://192.168.0.110:32576/](https://192.168.0.110:32576/)访问到 Dashboard(Chrome 可能会提示证书错误,无法访问,Fix)

![Dashboard](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/KubernetesDashboard-1.png)

### 创建 ServiceAccount

```bash
vi admin-role.yaml
```

输入以下内容

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

```bash
kubectl apply -f admin-role.yaml
```

### 获取 Token

执行：

```bash
kubectl -n kubernetes-dashboard  get secret|grep admin-token
```

```bash
admin-token-r8b4b                        kubernetes.io/service-account-token   3      48m
kubernetes-dashboard-admin-token-qlnhp   kubernetes.io/service-account-token   3      60m
```

执行：

```bash
kubectl -n kubernetes-dashboard describe secret admin-token-r8b4b
```

```bash
Name:         admin-token-r8b4b
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 03a2bca0-b6c0-4cde-93aa-c4a6cd70dfdb

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1yOGI0YiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjAzYTJiY2EwLWI2YzAtNGNkZS05M2FhLWM0YTZjZDcwZGZkYiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbiJ9.g_dtJjhbLVfJRcdhlyYH-ekn08Dv3_Ok9oMZ7o0jU0Ri90sIhaANaprVlGK7QiKzIkz_BNT1Hw_reAseoOy7smFriKhn4a4wPMO0Ir1aJPavDdoVIEhBDHHzrukXl3mVO92WgkBkAMIo8HoVve-1pj9QVtT7hu_e8GXifyLu1v6s26lMbVouG8cPD4hzM2grRfhCt7qjioP3Gs6khtmHysu_uCBNW63HvuwzMBRS-lSr1ewWld4QnrvgqJ-IfLqAcjHjysNR26Xi9IBAswkq0E-1qSgIyduALITXx9FK9RqNBOTZ33OeDBCE-OYqmlIItDuYl4qRaksV3mccL4RVWA
```

将获取到的 Token 输入到 Dashboard 的输入框中，登录即可

![DashboardAfterLogin](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/KubernetesDashboard-2.png)

----------

## 遇到的问题

#### 1. 访问页面提示ServiceUnavailable

```bash
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "no endpoints available for service \"https:kubernetes-dashboard:\"",
  "reason": "ServiceUnavailable",
  "code": 503
}
```

查看 Dashboard Pod 的状态

```bash
kubectl get pods -n kube-system | grep dashboard
kubernetes-dashboard-77fd78f978-zqbs4   0/1     ImagePullBackOff   0          115m
```

查看 Pod 详细信息

```bash
kubectl -n kube-system describe pod kubernetes-dashboard-77fd78f978-zqbs4
Name:               kubernetes-dashboard-77fd78f978-zqbs4
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
Node:               ubuntu/192.168.111.129
Start Time:         Tue, 16 Oct 2018 09:50:14 +0000
Labels:             k8s-app=kubernetes-dashboard
                    pod-template-hash=77fd78f978
Annotations:        <none>
Status:             Pending
IP:                 10.32.0.4
Controlled By:      ReplicaSet/kubernetes-dashboard-77fd78f978
Containers:
  kubernetes-dashboard:
    Container ID:
    Image:         k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
    Image ID:
    Port:          8443/TCP
    Host Port:     0/TCP
    Args:
      --auto-generate-certificates
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Liveness:       http-get https://:8443/ delay=30s timeout=30s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /certs from kubernetes-dashboard-certs (rw)
      /tmp from tmp-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kubernetes-dashboard-token-7skvp (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  kubernetes-dashboard-certs:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kubernetes-dashboard-certs
    Optional:    false
  tmp-volume:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  kubernetes-dashboard-token-7skvp:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kubernetes-dashboard-token-7skvp
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason   Age                     From             Message
  ----     ------   ----                    ----             -------
  Warning  Failed   9m17s (x458 over 119m)  kubelet, ubuntu  Error: ImagePullBackOff
  Normal   BackOff  4m14s (x479 over 119m)  kubelet, ubuntu  Back-off pulling image "k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0"
```

最后一行可以看到在拉取镜像的时候失败了；可以先拉取镜像再启动，这里有两种解决办法：

```bash
# 1. 如果网络可以拉取到镜像，直接手动拉取即可
docker pull k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0

# 2. 如果拉取不到，尝试从其他镜像源拉取重新打标签
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0
```

拉取到镜像之后等待一会儿，Kubernetes 会自动创建新的 Pod；或者也可以删除 Dashboard 所有资源重新创建：

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

#### 2. 重启后使用  kubectl 提示 `The connection to the server 192.168.111.129:6443 was refused - did you specify the right host or port?`

重启Ubuntu 后，访问Dashboard timeout，通过`kubectl get pods -n kube-system`查看 Pod 状态，提示

```bash
The connection to the server 192.168.111.129:6443 was refused - did you specify the right host or port?
```

以为是配置的问题，但是参考 [https://github.com/kubernetes/kubernetes/issues/50295#issuecomment-376603921](https://github.com/kubernetes/kubernetes/issues/50295#issuecomment-376603921)，尝试后依然无法解决；最后尝试使用`kubeadm init`重新创建，提示

```bash
running with swap on is not supported. Please disable swap
```

因为 Swap 导致Kubenetes 没有成功启动，执行关闭 swap，重新启动后解决问题

```bash
sudo swapoff -a
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
