---
title: Kubenetes 中使用 Traefik 作为 Ingress 转发流量
date: 2019-09-08 19:06:11
tags:
    - Kubernetes
    - Helm
    - Traefik
    - Ingress
categories: 
    - Kubernetes
    - Helm
    - Traefik
    - Ingress
---

# Kubenetes 中使用 Traefik 作为 Ingress 转发流量

Ingress 就是 Kubernetes 机器外访问集群的入口，将请求的 URL 转发到不同的 Service 上，相当于 Nginx 等代理服务器
路由信息由 Ingress Controller 提供，Ingress Controller 可以理解为监视器，不断请求 Kubernetes API 实时感知 Service 和 Pod 的状态，结合上下文的 Ingress 生成配置，然后更新反向代理服务器的配置，达到服务发现的作用

Traefik 是一个开源的反向代理与负载均衡工具，能够与常见的微服务系统直接整合，可以实现自动化动态配置

## 通过配置文件部署 Traefik

### 配置

- Ingress-rbac.yaml  

用于 Service Account 验证

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress
  namespace: kube-system

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: ingress
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

- traefik-daemon-set.yaml

使用 DaemonSet 部署 Traefik

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress-lb
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      restartPolicy: Always
      serviceAccountName: ingress
      containers:
      - image: traefik
        name: traefik-ingress-lb
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8580
          hostPort: 8580
        args:
        - --web
        - --web.address=:8580
        - --kubernetes
```

- traefik-ui.yaml

创建 Traefik 的 UI

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  type: NodePort  
  ports:
  - name: web
    port: 80
    targetPort: 8580
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik-ui.local
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
```

### 部署

```bash
kubectl apply -f .
```

## 通过 Helm 部署 Traefik

- 创建 values.yaml 

```yaml
dashboard:
  enabled: true
  domain: dashboard.traefik
serviceType: LoadBalancer
rbac:
  enabled: true
deployment:
  hostPort:
    httpEnabled: true
    httpsEnabled: true
    dashboardEnabled: true
```

```bash
helm install --values values.yaml stable/traefik --name traefik
```

这样就会启用 Dashboard，更改 Host 指向相应的节点和端口，访问`dashboard.traefik:${PORT}`就可以看到 Dashboard

- Host

```bash
192.168.0.110 dashboard.traefik
```

- 也可以直接下载 Traefik 的 Helm Release，解压后修改相应的配置后再安装

```bash
helm fetch stable/traefik

tar -xvf traefik-1.24.1.tgz

helm install --values values.yaml ./traefik --name traefik
```

## 部署应用 

- backend-ingress.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: default
spec:
  rules:
  - host: rest.hellowoodes.com
    http:
      paths:
      - path: /
        backend:
          serviceName: backend
          servicePort: 8080
```

修改 Host 将域名指向相应的节点 IP


## 测试

- 获取端口

```bash
kubectl get service -l app=traefik
```

```bash
NAME               TYPE          CLUSTER-IP      EXTERNAL-IP  PORT(S)                     AGE
traefik            LoadBalancer  10.106.137.170  <pending>    80:32311/TCP,443:30677/TCP  1s
traefik-dashboard  ClusterIP     10.108.91.184   <none>       80/TCP                      1s
```

- 访问相应的 NodeIP 和地址 [http://dashboard.traefik:32311](http://dashboard.traefik:32311)，可以看到 Traefik 的 Dashboard

- 因为在values.yaml 中已经配置了 `deployment.hostPort.httpEnabled`和`deployment.hostPort.httpsEnabled`，所以也可以直接访问 [dashboard.traefik](http://dashboard.traefik)

![traefik-dashboard-page.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/traefik-dashboard-page.png)

- 访问backend 应用 

```bash
curl http://rest.hellowoodes.com/ping
Pong%
```

----------
 
## 参考资料

- [Helm Traefik](https://github.com/helm/charts/tree/master/stable/traefik)
- [Kubernetes Traefik Installation (helm)](https://8gwifi.org/docs/kube-traefik.jsp)
- [Kubernetes Ingress Controller](https://docs.traefik.io/user-guide/kubernetes/#deploy-traefik-using-helm-chart)
- [安装traefik ingress](https://jimmysong.io/kubernetes-handbook/practice/traefik-ingress-installation.html)
- [微服务边界路由Traefik：基于京东云Kubernetes集群及Helm Chart](https://blog.csdn.net/deepbluesolo/article/details/84901427)
