---
title: 使用Gradle整合Flyway进行数据库迁移
date: 2018-01-01 00:49:34
tags:
    - Java
    - Gradle
    - Flyway
categories: 
    - Java
    - Gradle
    - Flyway
---
> 使用Flyway进行数据库迁移可以极大的减少开发过程中对数据库版本的操作，使用Gradle整合Flyway可以更好的和项目契合

## 配置build.gradle文件

```
apply plugin: 'org.flywaydb.flyway'

buildscript {
    repositories {
        mavenCentral()
    }

    dependencies {
        classpath(group: 'org.flywaydb', name: 'flyway-gradle-plugin', version: "4.0.3")
        classpath(group: 'mysql', name: 'mysql-connector-java', version: "5.1.41")
    }
}

flyway {
    url = 'jdbc:mysql://localhost:3306/miniprograme?useSSL=false'
    locations = ['filesystem:db/migration']
    user = 'root'
    password = '123456'
    schemas = ['flywaydb']
}
```

##使用Flyway进行数据库管理


- 删除数据库所有表
```
gradle flywayClean
```
- 迁移数据库
```
gradle flywayMigrate
```


- 校验新版本文件是否有冲突
```
gradle flywayValidate
```

- 查看数据库状态
```
gradle flywayInfo
```

- 修复数据库（删除失败的版本，修复checksum值）
```
gradle flywayRepair
```
- 设置某一版本为基础版本，该版本及之前的不会再被执行
```
gradle flywayBaseline
```

