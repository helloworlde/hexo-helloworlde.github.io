---
title: Kubernetes 配置 kubeconfig 访问多个集群
date: 2018-10-23 21:09:48
tags:
    - Kubernetes
categories: 
    - Kubernetes
---

# Kubernetes 配置 kubeconfig 访问多个集群

> 如果有多个不同的集群，需要切换访问，就需要配置多个 Kubernetes 账号和 Context；集群的 KubeConfig 文件一般为`~/.kube/config`，默认只能访问一个集群，如果需要访问多个集群就需要修改这个文件

> 可以参考文档 [https://kubernetes.io/zh/docs/tasks/access-application-cluster/configure-access-multiple-clusters/](https://kubernetes.io/zh/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)配置，但是这个文档描述不够直接简单，可以参考以下内容直接修改

假设现在有两个集群，一个是本地的 `docker-for-desktop-cluster`，另一个是部署在测试环境的 `kubernetes` ，本地只有 `docker-for-desktop-cluster`的配置 

根据官方文档合并后的 Demo，其实是将两个 Config 文件的相同类型的字段直接合并了，所以直接将相同的字段的其他集群的配置内容复制到当前的配置中即可，如：将`kubernetes`中 `cluster` 的配置直接复制到当前配置中：

- 测试环境 cluster 配置

```
- cluster:
    certificate-authority-data: CERTIFICATE_AUTHORITY_DATA 
    server: https://192.168.111.129:6443
  name: kubernetes

```

- 本地集群  cluster 配置

```
- cluster:
    insecure-skip-tls-verify: true
    server: https://localhost:6443
  name: docker-for-desktop-cluster
```

- 合并后

```
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://localhost:6443
  name: docker-for-desktop-cluster
- cluster:
    certificate-authority-data: CERTIFICATE_AUTHORITY_DATA
    server: https://192.168.111.129:6443
  name: kubernetes
```

- 合并完成后更新配置

```
export KUBECONFIG=~/.kube/config
```

- 查看合并后的 kubeconfig 

```
kubectl config view
```

```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://localhost:6443
  name: docker-for-desktop-cluster
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.111.129:6443
  name: kubernetes
contexts:
- context:
    cluster: docker-for-desktop-cluster
    user: docker-for-desktop
  name: docker-for-desktop
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: docker-for-desktop
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

- 查看集群 

```
kubectl config get-contexts
```

```
CURRENT   NAME                          CLUSTER                      AUTHINFO             NAMESPACE
          docker-for-desktop            docker-for-desktop-cluster   docker-for-desktop
*         kubernetes-admin@kubernetes   kubernetes                   kubernetes-admin
```

- 切换集群

```
kubectl config use-context docker-for-desktop
kubectl config use-context kubernetes-admin@kubernetes
```

- 如果想限制用户的 Namespace，可以在 context 中加入namespaces 配置

```
- context:
    cluster: docker-for-desktop-cluster
    user: docker-for-desktop
    namespace: default
  name: docker-for-desktop
```

----------

- 本地`docker-for-desktop-cluster` 的配置 (`~/.kube/config`)内容

```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://localhost:6443
  name: docker-for-desktop-cluster
contexts:
- context:
    cluster: docker-for-desktop-cluster
    user: docker-for-desktop
  name: docker-for-desktop
current-context: docker-for-desktop
kind: Config
preferences: {}
users:
- name: docker-for-desktop
  user:
    client-certificate-data: CLIENT_CERTIFICATE_DATA
    client-key-data: CLIENT_KEY_DATA
```


- 测试环境 `kubernetes` 的 配置(`~/.kube/config`)内容

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: CERTIFICATE_AUTHORITY_DATA 
    server: https://192.168.111.129:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: CLIENT_CERTIFICATE_DATA
    client-key-data: CLIENT_KEY_DATA
```

- 合并后

```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://localhost:6443
  name: docker-for-desktop-cluster
- cluster:
    certificate-authority-data: CERTIFICATE_AUTHORITY_DATA
    server: https://192.168.111.129:6443
  name: kubernetes
contexts:
- context:
    cluster: docker-for-desktop-cluster
    user: docker-for-desktop
  name: docker-for-desktop
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: docker-for-desktop
  user:
    client-certificate-data: CLIENT_CERTIFICATE_DATA
    client-key-data: CLIENT_KEY_DATA
- name: kubernetes-admin
  user:
    client-certificate-data: CLIENT_CERTIFICATE_DATA
    client-key-data: CLIENT_KEY_DATA
```