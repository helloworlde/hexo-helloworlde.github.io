---
title: AngularJS配置Router(路由)提示Controller NaNunction/undefined错误
date: 2018-01-01 12:05:17
tags:
    - AngularJs
    - Router
categories: 
    - AngularJs
    - Router
---
>在配置Angular 路由的时候，和以往一样使用如下配置：

- router.js

```
var adminApp = angular.module('adminApp', ['oc.lazyLoad', 'ui.router']);
angular.element(document).ready(function () {
    angular.bootstrap(document, ['adminApp']);
});

adminApp.run(function ($rootScope, $state, $stateParams) {
    $rootScope.$state = $state;
    $rootScope.$stateParams = $stateParams;
});
adminApp.config(function ($stateProvider, $urlRouterProvider) {
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
        });
});
```
- dashboardController.js
```
angular.module("adminApp").controller('dashboardController', function ($scope) {
    console.log("dashboardController");
});
```
> 但是提示错误：

```
angular.min.js:118 Error: [ng:areq] http://errors.angularjs.org/1.5.8/ng/areq?p0=dashboardController&p1=not%20aNaNunction%2C%20got%20undefined
    at http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:6:412
    at sb (http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:23:18)
    at Pa (http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:23:105)
    at http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:89:310
    at Object.<anonymous> (http://localhost:63342/static/plugin/angular-1.5.8/angular-ui-router.js:3971:42)
    at http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:16:71
    at la (http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:81:90)
    at p (http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:66:341)
    at g (http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:58:481)
    at http://localhost:63342/static/plugin/angular-1.5.8/angular.min.js:58:119
```


----------


> 原因是因为直接写controller无法识别，所有需要使用register来注册该controller

- router.js
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
    
    //以下是新加入的
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
        });
});
```
- dashboardController.js
```
angular.module("adminApp").controllerProvider.register('dashboardController', function ($scope) {
    console.log("dashboardController");
});
```