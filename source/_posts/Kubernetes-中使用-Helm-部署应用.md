---
title: Kubernetes 中使用 Helm 部署应用
date: 2019-09-08 19:03:06
tags:
    - Kubernetes
    - Helm
categories: 
    - Kubernetes
    - Helm
---

# Kubernetes 中使用 Helm 部署应用

## 创建应用

创建一个简单的应用，提供一个 REST 接口；使用 Golang 编写，然后将镜像 push 到 Docker Hub

- go.mod

```go
module github.com/helloworlde/rest

go 1.12
```

- main.go 

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/ping", func(writer http.ResponseWriter, request *http.Request) {
		fmt.Println("Pong")
		_, _ = fmt.Fprint(writer, "Pong")
	})

	fmt.Println("Server Started")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

- Dockerfile 

```dockerfile
FROM golang AS build-env
WORKDIR /go/src/github.com/hellowrolde/rest
COPY . .

ENV GO111MODULE on
ENV GOPROXY=https://gocenter.io
ENV GOOS linux
ENV GOARCH 386

RUN go mod download
RUN go build -v -o /go/src/github.com/hellowrolde/rest/app

FROM alpine
COPY --from=build-env  /go/src/github.com/hellowrolde/rest/app /usr/local/bin/app
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
EXPOSE 8080
CMD ["app"]
```

- 构建并 push 镜像

```bash
docker build -t hellowooeds/rest . 
docker push hellowooeds/rest
```

## 使用 Helm

### 添加 Helm

#### 初始化

```bash
helm create rest
```

然后会在项目目录下创建一个 rest 的文件夹，里面包含所需要的 Helm 配置文件

```bash
.
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 8 files
```

- Chart.yaml: 用于描述 Chart 的属性
- values.yaml: 存放项目的配置项，如镜像名称或者 tag 等，用于和模板组装
- charts: 用于存放依赖的 chart 的，当有依赖需要管理时，可以添加 requirements.yaml 文件，可用于管理项目内或者外部的依赖
- templates: 用于存放需要的配置模板


#### 修改配置文件

- Chart.yaml 

```yaml
apiVersion: v1
appVersion: "1.0"
description: Rest application with Helm
name: rest
version: 0.1.0
maintainers:
  - name: HelloWood
```

- 修改 values.yaml 

```yaml
backend:
  image: hellowoodes/rest
  tag: "latest"
  pullPolicy: IfNotPresent
  replicas: 1

namespace: rest

service:
  type: NodePort
```

- backend-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: backend
  name: backend
  namespace: {{ .Values.namespace }}
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  selector:
    app: backend
  type: {{ .Values.service.type }}
```

- backend-deployment.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: backend
  name: backend
  namespace: {{ .Values.namespace }}
spec:
  selector:
    matchLabels:
      app: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: {{ .Values.backend.image }}:{{ .Values.backend.tag }}
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

### 安装应用

```bash
helm install ./rest --name rest
```

如果不指定名称，会随机生成一个

```bash
NAME:   rest
LAST DEPLOYED: Sun Jul 14 18:50:16 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                      READY  STATUS             RESTARTS  AGE
backend-5bc9d6cb99-pfwn5  0/1    ContainerCreating  0         0s

==> v1/Service
NAME     TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)         AGE
backend  NodePort  10.105.41.183  <none>       8080:31381/TCP  0s

==> v1beta1/Deployment
NAME     READY  UP-TO-DATE  AVAILABLE  AGE
backend  0/1    1           0          0s


NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get service -l app=backend --namespace default  -o jsonpath="{.items[0].spec.ports[0].nodePort}")
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```

#### 查看 

- 查看 Helm 信息

```bash
helm list
```

```bash
NAME	REVISION	UPDATED                 	STATUS  	CHART     	APP VERSION	NAMESPACE
rest	1       	Sun Jul 14 18:55:25 2019	DEPLOYED	rest-0.1.0	1.0        	default
```

- 查看 Deployment、Service、Pod

```bash
kubectl get all -l app=backend
``` 

```bash
NAME                           READY   STATUS    RESTARTS   AGE
pod/backend-5bc9d6cb99-kmpvx   1/1     Running   0          7m15s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/backend      NodePort    10.104.248.37   <none>        8080:31064/TCP   7m15s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend   1/1     1            1           7m15s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-5bc9d6cb99   1         1         1       7m15s
```

#### 测试 

访问通过 NOTES 获取到的 IP 和端口

```bash
http get http://192.168.0.110:31381/ping
HTTP/1.1 200 OK
Content-Length: 4
Content-Type: text/plain; charset=utf-8
Date: Sun, 14 Jul 2019 10:53:18 GMT

Pong
```

### 打包 

```bash
helm package ./rest
```

```bash
Successfully packaged chart and saved it to: /path/to/project/rest-0.1.0.tgz
```

### 更新

当应用发生了更新之后，需要升级部署，这时候可以通过修改 Helm 的配置来实现；如镜像版本从1.0 更新到 1.1

- 修改 values.yaml

```yaml
backend:
  image: hellowoodes/rest
  tag: "1.1"
  pullPolicy: IfNotPresent
  replicas: 1

namespace: default

service:
  type: NodePort
```

- 执行更新 

```bash
 helm upgrade rest ./rest --repo local
```

```bash
Release "rest" has been upgraded. Happy Helming!
LAST DEPLOYED: Sun Jul 14 21:03:57 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                     READY  STATUS   RESTARTS  AGE
backend-56876f876-vvbzz  1/1    Running  0         85s

==> v1/Service
NAME     TYPE      CLUSTER-IP   EXTERNAL-IP  PORT(S)         AGE
backend  NodePort  10.97.22.28  <none>       8080:31496/TCP  7m10s

==> v1beta1/Deployment
NAME     READY  UP-TO-DATE  AVAILABLE  AGE
backend  1/1    1           1          7m10s


NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get service -l app=backend --namespace default -o jsonpath="{.items[0].spec.ports[0].nodePort}")
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```

等待更新完成后，查看 Pod 的描述信息

```bash
kubectl describe pod backend-56876f876-vvbzz
```

可以看到 pod 的镜像已经从 1.0 更新到了 1.1 

```bash
Normal  Pulled     29s   kubelet, ubuntu-server  Container image "hellowoodes/rest:1.1" already present on machine
```

### 

### 删除应用

删除 Helm 应用后，所有相关的 Service，Deployment，Pod 等都会被删除

```bash
helm delete --purge rest
```

### 查看 Repo 应用

```bash
helm serve
```

```bash
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879
```

![helm-app-list.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/helm-app-list.png)

----------

### 项目地址

[https://github.com/helloworlde/helm-app-rest](https://github.com/helloworlde/helm-app-rest)
