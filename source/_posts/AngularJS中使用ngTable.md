---
title: AngularJS中使用ngTable
date: 2018-01-01 11:42:21
tags:
    - AngularJs
    - ngTable
categories: 
    - AngularJs
    - ngTable
---
>在HTML中使用[ngTable](http://ng-table.com/) 可以方便的进行排序，筛选，分页，添加，编辑删除等操作，不用再从数据库里面进行分页等操作

- 需要引用的文件
    - angular.js
    - ng-table.js
    - ng-table.css
    - bootrasp.css


----------
- 注入依赖
-  为ng-table 设置属性和数据
```
var app = angular.module('app', [ 'ngTable']);
app.controller('controller', function($scope,NgTableParams) {
    $http.get("/query")
        .success(function(data) {
            $scope.userTable = new NgTableParams({
                        page : 1,
                        count : 5
                    }, {
                        total : data.length,
                        getData : data
                        }
                    }); 
            })
        });
}
```

- 为 table 添加 ng-table 属性
- 显示数据

```
<table ng-table="userTable" class="table table-hover">
            <tbody>
                <tr>
                    <th>编号</th>
                    <th>用户名</th>
                </tr>
                <tr ng-repeat="user in $data">
                    <td>{{user.id}}</td>
                    <td>{{user.name}}</td>
                </tr>
            </tbody>
        </table>
```