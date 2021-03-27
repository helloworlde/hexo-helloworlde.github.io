---
title: AngularJS 中ngTable自定义/重写过滤器
date: 2018-01-01 11:56:49
tags:
    - AngularJs
    - ngTable
categories: 
    - AngularJs
    - ngTable
---
>- 在使用ngTable 时用到了需要进行按时间过滤，但是ngTable并没有该功能，所以需要自定义过滤器，但是如果自定义了过滤器，则会覆盖原来的，所以就需要重写过滤器

>- ###ngTable过滤器的原理是按照过滤的条件遍历所有的列表项内容，如果满足过滤条件则返回true，显示该记录，如果不满足条件则返回false，不显示该条记录，过滤条件有任何变化都会触发过滤

> - text:根据字符匹配，如果被过滤的值有该字符，则显示该记录
- number:根据数值进行匹配，如果数值相等，则显示该记录
- select:根据下拉列表选择的值进行匹配，如果值相等，则显示该记录

- 添加自定义的过滤器控件
    - 在HTML中
```
<script type="text/ng-template" id="/filter/js/startDateFilter.html">
    <input type="text">
</script>

```
- 添加自定义的过滤器控件 
    - 在js中
    

```
$scope.customFilter = {
            start:{
                id:'/filter/js/startDateFilter.html',
                placeholder:'Start'
            }
        }
```

- 需要在ngTable的配置中指定过滤器

```
$scope.userTable = new NgTableParams({},
            {
                // initial sort order
                filterDelay: 0,
                filterOptions: {filterFn: $scope.customFilterFn},
                dataset: data
            });
```

- 为指定列设置控件

```
<td data-title="'日期'" filter="customFilter">
                    {{user.date|date:'yyyy-MM-dd'}}
                </td>
```

- 重写过滤器方法，实现过滤器功能

```
$scope.customFilterFn = function (date, filterValues) {
            return data.filter(function (item) {
                var result = true;
                if (typeof(filterValues.name) != undefined && filterValues.name != null) {
                    result = result && (item.name.indenOf(filterValues.name) > -1);
                }

                if (typeof(filterValues.sex) != undefined && filterValues.sex != null) {
                    result = result && (item.sex == filterValues.sex);
                }

                if (typeof(filterValues.startDate) != undefined && filterValues.startDate != null) {
                    result = result && (item.startDate >= filterValues.startDate);
                }
                return result;
            })
        }
```


----------
- 页面代码

```
<!DOCTYPE html>
<html lang="en" ng-app="myApp" ng-controller="myCtrl">
<head>
    <meta charset="UTF-8">
    <title>自定义过滤器</title>
</head>
<link href="https://cdn.bootcss.com/bootstrap/4.0.0-alpha.6/css/bootstrap.min.css" rel="stylesheet">
<link href="https://cdn.bootcss.com/ng-table/1.0.0/ng-table.min.css" rel="stylesheet">
<body>
<div class="row">
    <!--自定义filter控件-->
    <script type="text/ng-template" id="dateFilter.html">
        <input type="date">
    </script>
    <div class="col-md-12 table-responsive">
        <table ng-table="userTable" class="table table-condensed table-bordered table-striped table-hover "
               show-filter="true">
            <tr ng-repeat="user in $data">
                <td data-title="'用户名'" filter="{username: 'text'}" sortable="'username'">{{user.username}}</td>
                <td data-title="'性别'" filter="{sex: 'select'}" filter-data="sexs" sortable="'sex'">{{user.sex}}</td>
                <td data-title="'性别'" filter="customFilter" sortable="'date'">
                    {{user.date|date:'yyyy-MM-dd'}}
                </td>
            </tr>
        </table>
    </div>
</div>
</body>
<script type="text/ng-template" id="/filter/js/startDateFilter.html">
    <input type="text">
</script>
<script src="https://cdn.bootcss.com/angular.js/1.5.8/angular.min.js"></script>
<script src="https://cdn.bootcss.com/ng-table/1.0.0/ng-table.min.js"></script>

<script type="text/javascript">
    var app = angular.module('myApp', ['ngTable']);
    app.controller('myCtrl', function ($scope, NgTableParams) {

        $scope.sexs = [{
            "id": "男",
            "title": "男"
        }, {
            "id": "女",
            "title": "女"
        }]

        var data = [{
            "username": "农夫山泉",
            "sex": "男",
            "date": 1004570428580
        }, {
            "username": "哇哈哈",
            "sex": "女",
            "date": 1784570428580
        },
            {
            "username": "Alice",
            "sex": "男",
            "date": 1466570428580
        },
            {
            "username": "CCC",
            "sex": "女",
            "date": 1584570428580
        }];


        $scope.userTable = new NgTableParams({
                sorting: {id: "asc"}
            },
            {
                // initial sort order
                filterDelay: 0,
                filterOptions: {filterFn: $scope.customFilterFn},
                dataset: data
            });

        $scope.customFilter = {
            start:{
                id:'/filter/js/startDateFilter.html',
                placeholder:'Start'
            }
        }

        $scope.customFilterFn = function (date, filterValues) {
            return data.filter(function (item) {
                var result = true;

                console.log(filterValues);

                if (typeof(filterValues.name) != undefined && filterValues.name != null) {
                    result = result && (item.name.indenOf(filterValues.name) > -1);
                }

                if (typeof(filterValues.sex) != undefined && filterValues.sex != null) {
                    result = result && (item.sex == filterValues.sex);
                }

                if (typeof(filterValues.startDate) != undefined && filterValues.startDate != null) {
                    result = result && (item.startDate >= filterValues.startDate);
                }
                return result;
            })

        }
    })
</script>
</html>
```