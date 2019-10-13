---
title: 使用腾讯云的Serverless部署天气应用
date: 2019-10-13 18:56:40
    - Serverless
    - Go
categories: 
    - Serverless
    - Go
---

# 使用腾讯云的Serverless部署应用

> 使用腾讯云的Serverless服务，部署一个Go编写的天气变化的提醒应用
> 该应用通过定时查询高德地图的[天气API](https://lbs.amap.com/api/webservice/guide/api/weatherinfo)，当当前天气或未来几天天气不好时，通过[Server酱](http://sc.ftqq.com)在微信中进行提醒

## 构建应用

应用使用 go modules开发

- go.mod

```go
module weather

go 1.12

require github.com/tencentyun/scf-go-lib v0.0.0-20190817080819-4a2819cda320
```

- main.go

```go
package main

import (
	"log"
	"os"
	"strconv"
	"time"

	"fmt"

	"github.com/tencentyun/scf-go-lib/cloudfunction"
	"weather/tool"
)

func main() {
	cloudfunction.Start(checkWeather)
}

func checkWeather() (string, error) {
    // ...
}
```

## 创建函数

在腾讯云的[Serverless服务](https://console.cloud.tencent.com/scf/list?rid=8&ns=default)中创建新的函数

![创建函数](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9oZWxsb3dvb2Rlcy5vc3MtY24tYmVpamluZy5hbGl5dW5jcy5jb20vYmxvZy9xY2xvdWQtc2VydmVybGVzcy8lRTUlODglOUIlRTUlQkIlQkElRTUlODclQkQlRTYlOTUlQjAucG5n?x-oss-process=image/format,png)

## 添加配置 

配置共三项：

- 高德地图的SecretKey，可以在高德地图的[控制面板](https://lbs.amap.com/dev/key/app)中添加应用后获取
- Server酱的SecretKey，在发送的URL中可以找到
- 城市id，高德地图的城市id，可以在[城市编码](https://lbs.amap.com/api/webservice/download)中找到

### 添加环境变量

在函数配置点击编辑，添加环境变量

```
city       xxxx
weatherKey xxxx
notifyKey  xxxx
```

## 上传函数

### 本地编译打包

- Mac/Linux

```bash
GOOS=linux GOARCH=amd64 go build -o main main.go
zip main.zip main
```

- Win 

```bash
set GOOS=linux
set GOARCH=amd64
go build -o main main.go
```

然后将main添加到压缩文件中

### 上传zip

在函数代码中上传压缩文件，保存

![上传函数](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9oZWxsb3dvb2Rlcy5vc3MtY24tYmVpamluZy5hbGl5dW5jcy5jb20vYmxvZy9xY2xvdWQtc2VydmVybGVzcy8lRTQlQjglOEElRTQlQkMlQTAlRTUlODclQkQlRTYlOTUlQjAucG5n?x-oss-process=image/format,png)

### 测试

待上传完成后，选择 HelloWorld测试模板，点击测试，等函数执行，会看到测试结果

![测试函数](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9oZWxsb3dvb2Rlcy5vc3MtY24tYmVpamluZy5hbGl5dW5jcy5jb20vYmxvZy9xY2xvdWQtc2VydmVybGVzcy8lRTYlQjUlOEIlRTglQUYlOTUlRTUlODclQkQlRTYlOTUlQjAucG5n?x-oss-process=image/format,png)

### 添加触发方式

在触发方式中添加定时触发，cron表达式为 `0 30 8-21 * * * *`，这样就可以在每天8点-21点的30分触发查询，如果天气状况不佳，会通过微信通知


--------


### 参考文档

- [开发语言 Golang](https://cloud.tencent.com/document/product/583/18032)
- [天气查询](https://lbs.amap.com/api/webservice/guide/api/weatherinfo)


### 项目地址

- [https://github.com/helloworlde/weather](https://github.com/helloworlde/weather)
