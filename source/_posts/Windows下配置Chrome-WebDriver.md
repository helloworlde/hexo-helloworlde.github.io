---
title: Windows下配置Chrome WebDriver
date: 2018-01-01 11:59:09
tags:
    - WebDriver
    - Selenium
categories: 
    - WebDriver
    - Selenium
---
> WebDriver多用来执行自动化测试，可以通过Java文件或者其他方式在测试的时候打开，Firefox的自带了WebDriver，但是Chrome没有，需要手动安装

------------------------

- 首先下载[Chrome的WebDriver](https://sites.google.com/a/chromium.org/chromedriver/downloads)
- 将WebDriver复制到Chrome的安装目录
    - 安装目录可以通过在Chrome地址栏中输入`chrome://version/`来查看
    - 一般默认的安装目录是 `C:\Program Files (x86)\Google\Chrome`
    - 即将`chromedriver.exe`文件复制到`C:\Program Files (x86)\Google\Chrome\Application`下
- 将WebDriver的路径复制到系统环境变量PATH中
    - 即将`C:\Program Files (x86)\Google\Chrome\Application\chromedriver.exe`添加到PATH中


----------


这样就完成了Chrome WebDriver的配置