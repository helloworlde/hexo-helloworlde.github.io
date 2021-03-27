---
title: Gradle 使用-编译时下载依赖失败解决方法
date: 2018-01-01 01:00:20
tags:
    - Gradle
categories: 
    - Gradle
---
> 如果`Gradle`在编译的时候没有在本地仓库中发现依赖，就会从远程仓库中下载，默认的远程仓库为`mavenCentral()`，也就是[http://repo1.maven.org/maven2/](http://repo1.maven.org/maven2/)，但是往往访问速度特别慢，不翻墙经常会下载超时或者需要很长时间导致Build失败，因此，可以用国内的仓库代替：

> -  阿里的仓库地址：[http://maven.aliyun.com/nexus/content/groups/public/](http://maven.aliyun.com/nexus/content/groups/public/)
> - OSChina的仓库地址：[http://maven.oschina.net/content/groups/public/](http://maven.oschina.net/content/groups/public/)


----------

##修改单独项目



在项目的`build.gradle`文件中，修改`repositories`配置
由：

```
    repositories {
        mavenCentral()
    }
```
改为：

```
repositories {
        maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
    }
```
或：

```
repositories {
        maven{ url 'http://maven.oschina.net/content/groups/public/'}
    }
```
这样就可以从国内的镜像中下载依赖，速度能提高不少

> 注意： 
> 
> - build.gradle文件里有两处repositories，都需要改掉
> - 阿里云的速度比较快，更稳定

----------

## 更改所有项目
如果想一次更改所有的仓库地址，可以在`USER_HOME/.gradle/`（如`C:\Users\hellowood\.gradle`）文件夹下添加`init.gradle`文件来配置

- init.gradle
```
allprojects{
    repositories {
        def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public/'
        all { ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    remove repo
                }
            }
        }
        maven {
            url REPOSITORY_URL
        }
    }
}
```
这样就可以在项目编译的时候从阿里的仓库中下载依赖了

- `init.build`脚本可以参考[https://docs.gradle.org/current/userguide/init_scripts.html](https://docs.gradle.org/current/userguide/init_scripts.html)