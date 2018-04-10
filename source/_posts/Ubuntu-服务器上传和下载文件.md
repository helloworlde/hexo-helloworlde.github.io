---
title: Ubuntu 服务器上传和下载文件
date: 2018-04-10 14:47:07
tags:
    - Ubuntu
categories:   
    - Ubuntu
---

# Ubuntu 服务器上传和下载文件

使用 `scp` 命令完成文件的上传和下载 

## 上传

- 上传单个文件 

```
scp -p port source_dictionary_file user@ServerIp:target_dictionary_file
```

> prot 默认是22，如果使用默认可以不写

```
scp /User/hellowood/wechat.jpg root@192.168.0.2:/home/hellowood/wechat.jpg
```

- 上传整个文件夹 

```
scp -p port -r source_dictionary user@ServerIp:target_dictionary
```

```
scp -r /User/hellowood/tomcat root@192.168.0.2:/home/hellowood/tomcat
```

## 下载

- 下载单个文件

```
scp -p user@ServerIp:source_dictionary_file target_dictionary
```

```
scp root@192.168.0.2:/home/hellowood/wechat.jpg /User/hellowood/wechat.jpg
```

- 下载文件夹

```
scp -p port -r user@ServerIp:source_dictionary target_dictionry
```

```
scp -r root@192.168.0.2:/home/hellowood/tomcat /User/hellowood/tomcat
```