---
title: SpringBoot 使用 MySQL保存emoji 表情
date: 2018-12-31 22:59:16
tags:
    - MySQL
    - SpringBoot 
categories: 
    - MySQL
    - SpringBoot
---

# SpringBoot 使用 MySQL保存emoji 表情

> 在使用 SpringBoot 开发的应用中，有表单提交的内容中含有 emoji 表情，导致保存失败；这是因为MySQL 默认的 utf8 长度为3位，emoji 表情有4位

- 更改表的字符集为`utf8mb4`

```sql
ALTER TABLE content DEFAULT CHARACTER SET utf8mb4 COLLATE utf8_general_ci;
```

但是这样更改后仍然没有解决保存失败的问题

- 更改数据库的字符集

```sql
ALTER DATABASE db_name DEFAULT CHARACTER SET utf8mb4 COLLATE utf8_general_ci;
```

由于暂时不能重启数据库，这种修改方式只在当前连接中生效，同样未能解决问题

- 修改 SpringBoot 的配置 

```dsconfig
spring.datasource.hikari.connection-init-sql=SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci
```

这样，在应用的所有连接内都使用utf8mb4 作为字符集，解决了 emoji 保存失败的问题