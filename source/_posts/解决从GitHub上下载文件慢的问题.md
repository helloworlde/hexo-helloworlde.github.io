---
title: 解决从GitHub上下载文件慢的问题
date: 2018-01-01 11:34:48
tags:
    - GitHub
    - Experience
categories: 
    - GitHub
    - Experience
---
> ## 解决从GitHub上下载文件慢的问题
> 从GitHub下载文件一直非常慢，查看下载链接发现最终被指向了Amazon的服务器，下载地址是[http://github-cloud.s3.amazonaws.com/](http://github-cloud.s3.amazonaws.com/)，从国内访问Amazon非常慢，所以总是下载失败，解决方法时更改host文件，使该域名指向香港的服务器：
> ## 更改hosts文件：
> 更改`C:\Windows\System32\drivers\etc\hosts`文件，在文件中追加
> `219.76.4.4 github-cloud.s3.amazonaws.com`, 将域名指向该IP即可