---
title: Ubuntu搭建Redis服务器
date: 2018-01-01 12:08:07
tags:
    - Ubuntu
    - Reids 
categories: 
    - Ubuntu
    - Reids 
---
> 在Ubuntu中搭建Redis服务器

## 安装
###1 安装 

```
sudo apt-get install redis-server
```

###2 启动

```
redis-server
```

```
5617:C 18 Sep 12:57:10.437 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
5617:M 18 Sep 12:57:10.437 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.0.6 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 5617
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

5617:M 18 Sep 12:57:10.438 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
5617:M 18 Sep 12:57:10.438 # Server started, Redis version 3.0.6
5617:M 18 Sep 12:57:10.438 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
5617:M 18 Sep 12:57:10.438 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
5617:M 18 Sep 12:57:10.438 * DB loaded from disk: 0.000 seconds
5617:M 18 Sep 12:57:10.438 * The server is now ready to accept connections on port 6379
```

###3 启动客户端

```
redis-cli
```

```
127.0.0.1:6379> ping
PONG
```

###4 检查 Redis 服务器系统进程

```
ps -aux|grep redis
```

```
ubuntu    5617  0.0  0.7  38856  7884 pts/0    Sl+  12:57   0:01 redis-server *:6379
ubuntu    5707  0.0  0.0  12944   976 pts/1    S+   13:22   0:00 grep redis
```

###5 通过端口检查 Redis 服务器状态

```
netstat -nlt|grep 6379
```

```
tcp        0      0 0.0.0.0:6379            0.0.0.0:*               LISTEN     
tcp6       0      0 :::6379                 :::*                    LISTEN  
```

###6 通过启动命令检查 Redis 服务器状态

```
sudo /etc/init.d/redis-server status
```

```
redis-server is running
```
-----------------

## 配置
###1 设置登录密码

```
# 取消注释requirepass
requirepass your_password
```

###2 取消登录IP限制

```
# 注释bind
# bind 127.0.0.1
```