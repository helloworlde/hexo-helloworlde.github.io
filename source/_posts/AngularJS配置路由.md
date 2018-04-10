---
title: AngularJS 配置路由
date: 2018-01-01 12:04:04
tags:
    - AngularJs
    - Router
categories: 
    - AngularJs
    - Router
---
> 在使用AngularJS的时候需要用到路由来控制页面的跳转，从而达到使用一个面板进行控制的目的，面板页面如图所示

![控制面板](http://img.blog.csdn.net/20170523210031963?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzM2MDg1MA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 该面板分为菜单栏和控制页面两部分，左侧和上方为不变的部分，中间区域随菜单选择变动

----------
##[项目下载](http://download.csdn.net/detail/u013360850/9850266) | [GitHub下载](https://github.com/helloworlde/AngularRouter) | [演示地址](http://119.29.99.89/AngularRouter/pages/admin/index.html) | [GitHub演示地址](https://helloworlde.github.io/AngularRouter) 

----------


## 1.引入所需的CSS和JS文件
- 将所需要的CSS文件和JS文件引入到项目中index.html
 -  angular.min.js
 - ocLazyLoad.js
 - angular-ui-router.js
 
```
<!DOCTYPE html>
<html lang="en" class="body">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, height=device-height, initial-scale=1, maximum-scale=1"/>
    <link href="../../plugin/bootstrap-3.3.7/css/bootstrap.min.css" rel="stylesheet">
    <link href="dashboard/dashboard.css" rel="stylesheet">
</head>
<body >
<div data-ui-view="" style="height: 100%;overflow: hidden;"></div>
<script src="../../plugin/angular-1.5.8/angular.min.js"></script>
<script src="../../plugin/angular-1.5.8/angular-ui-router.js"></script>
<script src="../../plugin/oclazyload/ocLazyLoad.js"></script>
<script src="app.route.js"></script>
</body>
</html>
```

## 2. 配置dashboard页面

```
<div class="app app-header-fixed" id="app">
    <!-- navbar -->
    <div class="app-header navbar">
        <div class="collapse pos-rlt navbar-collapse box-shadow bg-white-only">
            <ul class="nav navbar-nav navbar-right">
                <li class="dropdown">
                    <a href="#" data-toggle="dropdown" class="dropdown-toggle clear">
                      <span class="thumb-sm avatar pull-right m-t-n-sm m-b-n-sm m-l-sm">
                        <img src="/static/images/head.jpg">
                        <i class="on md b-white bottom"></i>
                      </span>
                        <span class="hidden-sm hidden-md">Administrator</span> <b class="caret"></b>
                    </a>
                    <!-- dropdown -->
                    <ul class="dropdown-menu animated fadeInRight w hidden-folded" role="menu">
                        <li>
                            <a href="#" role="menuitem">退出</a>
                        </li>
                    </ul>
                </li>
            </ul>
        </div>
    </div>

    <!-- menu -->
    <div class="app-aside hidden-xs bg-dark">
        <div class="aside-wrap">
            <div class="navi-wrap">
                <!-- nav -->
                <nav ui-nav class="navi">
                    <ul class="nav">
                        <li>
                            <a ui-sref=".accountManagement" class="auto">
                                <i class="glyphicon glyphicon-user"></i>
                                <span class="font-bold">账户管理</span>
                            </a>
                        </li>
                        <li>
                            <a ui-sref=".academicYear">
                                <i class="glyphicon glyphicon-calendar"></i>
                                <span class="font-bold">学年学期管理</span>
                            </a>
                        </li>
                        <li>
                            <a ui-sref=".feedbackManagement">
                                <i class="glyphicon glyphicon-send"></i>
                                <span class="font-bold">用户反馈管理</span>
                            </a>
                        </li>
                    </ul>
                </nav>
                <!-- nav -->
            </div>
        </div>
    </div>
    <!-- / menu -->
    <!-- content 中间可替换的部分-->
    <div class="app-content">
        <div data-ui-view="">
        </div>
    </div>
    <!-- /content -->
</div>
```

##3. 配置路由
- 配置各个页面的路由，通过懒加载加载所需要的文件
```
var adminApp = angular.module('adminApp', ['oc.lazyLoad', 'ui.router']);
angular.element(document).ready(function () {
    angular.bootstrap(document, ['adminApp']);
});

adminApp.run(function ($rootScope, $state, $stateParams) {
    $rootScope.$state = $state;
    $rootScope.$stateParams = $stateParams;
});
adminApp.config(function ($stateProvider, $urlRouterProvider, $controllerProvider) {
    adminApp.controllerProvider = $controllerProvider;
    $urlRouterProvider.when("", "dashboard/accountManagement");
    $urlRouterProvider.otherwise("dashboard/accountManagement");
    $stateProvider
        .state('dashboard', {
            url: '/dashboard',
            templateUrl: 'dashboard/dashboard.html',
            controller: 'dashboardController',
            resolve: {
                deps: ['$ocLazyLoad', function ($ocLazyLoad) {
                    return $ocLazyLoad.load(['dashboard/dashboard.js']);
                }]
            }
        })
        .state('dashboard.accountManagement', {
            url: '/accountManagement',
            templateUrl: 'accountManagement/accountManagement.html',
            controller: 'accountManagementController',
            resolve: {
                deps: ['$ocLazyLoad',
                    function ($ocLazyLoad) {
                        return $ocLazyLoad.load(['accountManagement/accountManagement.js', 'accountManagement/accountManagement.css']);
                    }]
            }
        })
        .state('dashboard.academicYear', {
            url: '/academicYear',
            templateUrl: 'academicYear/academicYear.html',
            controller: 'academicYearController',
            resolve: {
                deps: ['$ocLazyLoad',
                    function ($ocLazyLoad) {
                        return $ocLazyLoad.load(['academicYear/academicYear.js', 'academicYear/academicYear.css']);
                    }]
            }
        })
        .state('dashboard.feedbackManagement', {
            url: '/feedbackManagement',
            templateUrl: 'feedbackManagement/feedbackManagement.html',
            controller: 'feedbackManagementController',
            resolve: {
                deps: ['$ocLazyLoad',
                    function ($ocLazyLoad) {
                        return $ocLazyLoad.load(['feedbackManagement/feedbackManagement.js', 'feedbackManagement/feedbackManagement.css']);
                    }]
            }
        });
});
```

##4. 配置相应的页面controller

```
angular.module("adminApp").controllerProvider.register('dashboardController', function ($scope) {
    console.log("dashboardController");
});
```


