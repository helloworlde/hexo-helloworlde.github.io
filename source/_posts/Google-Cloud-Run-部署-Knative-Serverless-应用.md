---
title: 'Google Cloud Run 部署 Knative Serverless 应用 '
date: 2019-05-15 08:23:26
tags:
    - Serverless
    - Google
    - Cloud Run
    - Go
categories: 
    - Serverless
    - Google
    - Cloud Run
    - Go
---

# Google Cloud Run 部署Knative Serverless 应用 

> Google Cloud Run 是 Google 最近推出的基于容器运行的支持 Serverless 应用的服务，是 Knative 的Google Cloud 托管版本；和其他的 Serverless 如Google Cloud Functions, AWS Lambda 等相比，优点在于完全的基于容器，且不限语言

## 安装 [Cloud SDK](https://cloud.google.com/sdk/)

Cloud SDK 是 Google Cloud 的命令行工具，用于访问Google Cloud相关资源

具体平台的安装方式可以参考 [https://cloud.google.com/sdk/docs/quickstarts](https://cloud.google.com/sdk/docs/quickstarts)

## 创建应用，上传镜像

以 Go 语言为例，创建一个应用，根据不同的请求返回不同的内容

- main.go 

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
)

type CustomResponse struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
}

func main() {
	fmt.Println("Server started")
	http.HandleFunc("/", rootHandler)
	_ = http.ListenAndServe(":8080", nil)
}

func rootHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Println("Start handler request")

	queryForm, err := url.ParseQuery(r.URL.RawQuery)

	w.Header().Set("Content-Type", "application/json")
	message := ""

	if err == nil && len(queryForm["message"]) > 0 {
		message = queryForm["message"][0]
	} else {
		message = "Hello Go Server"
	}

	_ = json.NewEncoder(w).Encode(CustomResponse{200, message})
	fmt.Println("Handler request completed")
}
```

- Dockerfile

```dockerfile
FROM golang:1.12.3-alpine3.9
RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN CGO_ENABLED=0 GOOS=linux go build -o main main.go
EXPOSE 8080
CMD ["/app/main"]
```

- 配置 [Google Container Registry](https://console.cloud.google.com/gcr)

相关配置可以参考 [推送和拉取映像](https://cloud.google.com/container-registry/docs/pushing-and-pulling)，需要注意的是需要一个项目 ID，这个 ID 可以在 [home/dashboard](https://console.cloud.google.com/home/dashboard) 下找到

![Google Porject](http://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/GoogleCloudRun1.png)

- 配置本地 Docker

```bash
gcloud auth configure-docker
```

- 构建镜像 

```bash
docker build -t gcr.io/genial-post-128203/serverless .
```

- 推送镜像 

```bash
docker push gcr.io/genial-post-128203/serverless
```

## 创建 Serverless 应用

在[Cloud Run](https://console.cloud.google.com/run) 页面选择创建服务

![创建服务](http://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/GoogleCloudRun2.png)

![服务详情](http://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/GoogleCloudRun3.png)

## 测试

请求 URL [https://cloudserverless-pae2opltia-uc.a.run.app](https://cloudserverless-pae2opltia-uc.a.run.app)

- 不带参数

```bash
curl https://cloudserverless-pae2opltia-uc.a.run.app
```

```json
{"code":200,"message":"Hello Go Server"}
```

- 指定参数

```bash
curl https://cloudserverless-pae2opltia-uc.a.run.app?message=HelloWood
```

```json
{"code":200,"message":"HelloWood"}
```


----------

### 代码

- [https://github.com/helloworlde/google-cloud-run-go](https://github.com/helloworlde/google-cloud-run-go)

### 参考资料

- [Google Cloud Run详细介绍](https://skyao.io/post/201905-google-cloud-run-detail/)
- [推送和拉取映像](https://cloud.google.com/container-registry/docs/pushing-and-pulling?hl=zh-cn)
- [Using System Packages Tutorial](https://cloud.google.com/run/docs/tutorials/system-packages)