---
title: Kubernetes 中安装 Helm
date: 2019-09-08 19:00:28
tags:
    - Kubernetes
    - Helm
categories: 
    - Kubernetes
    - Helm
---

# Kubernetes 中安装 Helm

> Helm 是构建于 Kubernetes 之上的包管理器，可以理解为 yum，homebrew 或者 pip，用于简化包分发，安装，版本管理等操作流程

## 基本概念 

- Chart

chart 就是 Helm 所管理的包，包含着一个应用要部署至 Kubernetes 上所必须的所有资源

- Release

Release 就是 chart 在 Kubernetes 上部署后的实例，chart 每次部署都会产生一次 Release

- Repository

存储chart 的仓库，初始化 Helm 时会添加两个仓库，一个是 stable 仓库，地址是[https://kubernetes-charts.storage.googleapis.com/](https://kubernetes-charts.storage.googleapis.com/) ，另一个则是 local 仓库，地址是 [http://127.0.0.1:8879/charts](http://127.0.0.1:8879/charts)

- Config
Config 用于部署 chart 时自定义配置，在部署的时候，会将 Config 和 chart 进行合并，共同构成将部署的应用


## 安装 

Helm 是一个 C/S 架构，分为客户端helm 和服务端Tiller

### 客户端 

- Mac

```bash
brew install kubernetes-helm
```

- Ubuntu 

```bash
sudo snap install helm --classic
```

### 服务端

服务端安装要求 `$HOME/.kube/config`配置正确且有`kubectl`操作权限

- 创建账户 

tiller-rbac.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

```bash
kubectl apply -f tiller-rbac.yaml
```

- 安装 

```bash
helm init --service-account tiller
```

这种方式默认会使用 `gcr.io/kubernetes-helm/tiller`，可以通过指定镜像的方式初始化

```bash
helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.11.0 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts --service-account tiller
```

```bash
Creating /root/.helm
Creating /root/.helm/repository
Creating /root/.helm/repository/cache
Creating /root/.helm/repository/local
Creating /root/.helm/plugins
Creating /root/.helm/starters
Creating /root/.helm/cache/archive
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been upgraded to the current version.
```

- 查看版本

```bash
helm version
```

```bash
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"79d07943b03aea2b76c12644b4b54733bc5958d6", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
```

- 查看 deploy

```bash
kubectl -n kube-system get deploy tiller-deploy
```

```bash
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
tiller-deploy   1/1     1            1           91s
```

### 工作原理

helm 通过 gRPC 将 chart 发送至 Tiller，Tiller 则通过内置的 Kubernetes 客户端与Kubernetes 的API Server 进行交流，将 chart 进行部署，并生成 Release 用于管理

```bash
kubectl get svc -o wide -n kube-system

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)
tiller-deploy   ClusterIP   10.107.157.104   <none>        44134/TCP                24h    app=helm,name=tiller
```

Tiller 默认采用 ClusterIP 类型的 Service 进行部署，但是 ClusterIP 类型的 Service 仅限于集群内访问，所以 Helm 通过 socat 的端口转发，进而实现本地与 Tiller 的通信
