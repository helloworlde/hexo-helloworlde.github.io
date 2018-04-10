---
title: AngularJS中使用ngModal模态框
date: 2018-01-01 11:57:50
tags:
    - AngularJs
    - ngModal
categories: 
    - AngularJs
    - ngModal
---
- 在AngularJS中使用模态框需要引用的文件：
    - angular.js 1.5.5
    - ui.bootstrap-tpls.js 0.11.2
    - bootstrap.css 3.3.7

> 需要注意版本要一致，高版本的不支持这种方法，会出错


----------


- 将需要弹出的模态框的内容写在 script 标签中，指明属性，放在页面中

```
<script type="text/ng-template" id="modal.html"> 
<div>
    <div class="modal-header">
        <h3 class="modal-title" align="center">
            标题信息
        </h3>
    </div>
    <div class="modal-body">
        <div align="center">
            模态框内容
        </div>
    </div>
    <div class="modal-footer">
        <button class="btn btn-primary" ng-click="ok()">
            确认
        </button>
        <button class="btn btn-warning" ng-click="cancel()">
            退出
        </button>
    </div>
</div>
</script>
```

- 在App和Controller中注入模态框
```
var app = angular.module('app', ['ui.bootstrap']);
app.controller('modalController', function($scope, $rootScope,$modal) {
    $scope.openModel = function() {
            var modalInstance = $modal.open({
                templateUrl : 'modal.html',//script标签中定义的id
                controller : 'modalCtrl',//modal对应的Controller
                resolve : {
                    data : function() {//data作为modal的controller传入的参数
                         return data;//用于传递数据
                    }
                }
            })
        }
}

//模态框对应的Controller
app.controller('modalCtrl', function($scope, $modalInstance, data) {
    $scope.data= data;

    //在这里处理要进行的操作   
    $scope.ok = function() {
        $modalInstance.close();
    };
    $scope.cancel = function() {
        $modalInstance.dismiss('cancel');
    }
});
```

- 添加事件触发显示模态框

```
<button ng-click="openModal()">打开模态框</button>
```


----------
- html

```
<!DOCTYPE html>
<html ng-app="app" ng-controller="modalController">
<head>
    <title>ng-model模态框</title>
</head>
<link href="https://cdn.bootcss.com/bootstrap/3.3.7/css/bootstrap.min.css" rel="stylesheet">
<body>
<button ng-click="openModal()">打开模态框</button>

<script type="text/ng-template" id="modal.html"> 
    <div>
        <div class="modal-header">
            <h3 class="modal-title" align="center">
                标题信息
            </h3>
        </div>
        <div class="modal-body">
            <div align="center">
                模态框内容 <br>
                {{data}}
            </div>
        </div>
        <div class="modal-footer">
            <button class="btn btn-primary" ng-click="ok()">
                确认
            </button>
            <button class="btn btn-warning" ng-click="cancel()">
                退出
            </button>
        </div>
    </div>
</script>

<script src="https://cdn.bootcss.com/angular.js/1.5.5/angular.min.js"></script>
<script src="https://cdn.bootcss.com/angular-ui-bootstrap/0.11.2/ui-bootstrap-tpls.min.js"></script>

<script type="text/javascript">
    var app = angular.module('app', ['ui.bootstrap']);
    app.controller('modalController', function($scope, $rootScope, $modal) {
        var data = "通过modal传递的数据";
        $scope.openModal = function() {
                var modalInstance = $modal.open({
                    templateUrl : 'modal.html',//script标签中定义的id
                    controller : 'modalCtrl',//modal对应的Controller
                    resolve : {
                        data : function() {//data作为modal的controller传入的参数
                             return data;//用于传递数据
                        }
                    }
                })
            }
    })
     //模态框对应的Controller
     app.controller('modalCtrl', function($scope, $modalInstance, data) {
          $scope.data= data;

          //在这里处理要进行的操作
          $scope.ok = function() {
              $modalInstance.close();
          };
          $scope.cancel = function() {
              $modalInstance.dismiss('cancel');
          }
    });
</script>
</body>
</html>
```