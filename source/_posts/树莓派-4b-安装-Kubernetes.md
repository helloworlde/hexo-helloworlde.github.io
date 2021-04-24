---
title: 树莓派 4b 安装 Kubernetes
date: 2021-04-24 22:55:32
tags:
    - RaspberryPi
    - Kubernetes
    - K3S
    - Rancher
categories: 
    - RaspberryPi
    - Kubernetes
    - K3S
    - Rancher
---

# 树莓派 4b 安装 Kubernetes

K3S 是 Rancher 提供的用于边缘硬件的简化版本的 Kubernetes，基本能力和 Kubernetes 接近，适用于 IoT 硬件，支持 x86_64, ARMv7, ARM64 等

## 安装

在 Ubuntu Server 21.04 上安装 K3S 

### 1. 安装 Docker 

```bash
apt update & apt upgrade & apt install docker.io
```

### 2. 安装 K3S

登录树莓派所在的机器，执行安装脚本

```bash
curl -sfL https://get.k3s.io | sh -
```

安装完成后，使用 kubectl 查看集群信息

```bash
k3s kubectl get node

# 也可以直接使用 kubectl 
kubectl get node

NAME     STATUS   ROLES                  AGE   VERSION
ubuntu   Ready    control-plane,master   1h    v1.20.6+k3s1
```

### 3. 本地访问集群

- 获取 Kube Config 

默认的 Kubernetes Config 文件是 `/etc/rancher/k3s/k3s.yaml`，将该文件内容添加到本地，修改 server 的地址为树莓派的 IP 地址即可

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: xxx
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: xxx
    client-key-data: xxx
```

- 访问集群

```bash
kubectl get node --kubeconfig ~/.kube/pi

NAME     STATUS   ROLES                  AGE   VERSION
ubuntu   Ready    control-plane,master   9h    v1.20.6+k3s1
```

## 启动 Rancher

Rancher 是一个用于管理 K8S 集群的工具，可以在宿主机上通过 Docker 启动 Rancher，将集群导入管理

```bash
docker run -d --restart=unless-stopped -p 8080:80 -p 8443:443 rancher/rancher:v2.3.5
```

![RaspberryPiRancherCluster.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/RaspberryPiRancherCluster.png)


## 参考文档

- [k3s 官网](https://k3s.io/)
- [树莓派 4 使用 WiFi 从 SSD Headless 启动](https://helloworlde.github.io/2021/04/24/%E6%A0%91%E8%8E%93%E6%B4%BE-4b-%E4%BD%BF%E7%94%A8-WiFi-%E4%BB%8E-SSD-Headless-%E5%90%AF%E5%8A%A8/)
- [Kubenetes 部署 Dashboard](https://helloworlde.github.io/2019/09/08/Kubenetes-%E9%83%A8%E7%BD%B2-Dashboard/)
- [Kubernetes Dashboard](https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/)