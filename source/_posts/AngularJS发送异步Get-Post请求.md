---
title: AngularJS发送异步Get/Post请求
date: 2018-01-01 11:40:58
tags:
    - AngularJs
    - Request
categories: 
    - AngularJs
    - Request
---

 1 . 在页面中加入AngularJS并为页面绑定ng-app 和 ng-controller
 

```
<body ng-app="MyApp" ng-controller="MyCtrl" >
...
<script src="js/angular.min.js"></script>
<script src="js/sbt.js"></script>
```

 2 . 添加必要的控件并绑定相应的事件
 

```
    get:<input type="text" ng-model="param">{{param}} <br>
    post: <input type="text" ng-model="user.name"><input type="text" ng-model="user.password"><br>
    <button ng-click="get()">Get</button>
    <button ng-click="post()">Post</button>
```

 3 . 在JS脚本中发送进行Get/Post请求
  
  - get
 
```
$scope.get = function () {
        $http.get("/get", {params: {param: $scope.param}})
            .success(function (data, header, config, status) {
                console.log(data);
            })
            .error(function (data, header, config, status) {
                console.log(data);
            })
        ;
    }

```
  - get 将参数放在URL中
 
```
$scope.get = function () {
        $http.get("/get?param="+$scope.param)
            .success(function (data, header, config, status) {
                console.log(data);
            })
            .error(function (data, header, config, status) {
                console.log(data);
            })
        ;
    }

```

----------


- post 

```
$scope.post = function () {
        $http.post("/post", $scope.user)
            .success(function (data, header, config, status) {
                console.log(data);
            })
            .error(function (data, header, config, status) {
                console.log(data);
            })
        ;
    }
```

 4 . 由Controller处理请求并返回结果
 
- get
```
    @RequestMapping("/get")
    @ResponseBody
    public Map<String,String> get(String param) {
        System.out.println("param："+param);
        response.put("state", "success");//将数据放在Map对象中
        return response;
    }
```
- post

```
    @RequestMapping("/post2")
    @ResponseBody
    public void post2(@RequestBody  User user, HttpServletResponse resp) {
        //返回不同的http状态
        if(user.getName()!=null&&!user.getName().equals("")){
            resp.setStatus(200);
        }
        else{
            resp.setStatus(300);
        }
    }
```
- 如果需要配置请求头部

```
        $http({
            method : "POST",
            url : "/post",
            data : $scope.user
        }).success(function(data, header, config, status) {
            console.log(data);
        }).error(function(data, header, config, status) {
            console.log(data);
        });
```

 5 . 由JS http请求的回调函数处理并执行下一步操作


----------
- HTML

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Request</title>
</head>

<body ng-app="MyApp" ng-controller="MyCtrl" >
get:<input type="text" ng-model="param"><br>
post: <input type="text" ng-model="user.name"><input type="text" ng-model="user.password"><br>
    <button ng-click="get()">Get</button>
    <button ng-click="post()">Post</button>
</body>
<script src="js/angular.min.js"></script>
<script src="js/sbt.js"></script>
</html>
```
- sbt.js

```
var app = angular.module("MyApp", []);
app.controller("MyCtrl", function ($scope, $http) {

    $scope.get = function () {
        $http.get("/get", {params: {param: $scope.param}})
            .success(function (data, header, config, status) {
                console.log(data);
            })
            .error(function (response) {
                console.log(response);
            })
        ;
    }
   
    $scope.post = function () {
        $http.post("/post", $scope.user)
            .success(function (data, header, config, status) {
                console.log(data);
            })
            .error(function (data, header, config, status) {
                console.log(data);
            })
        ;
    }
});
```

