---
title: SpringMVC应用启动时执行操作
date: 2018-01-01 00:56:03
tags:
    - Java
    - SpringMVC 
categories: 
    - Java
    - SpringMVC
---

- ContextRefreshedEvent：当ApplicationContext初始化或者刷新时触发该事件。 

- ContextClosedEvent：当ApplicationContext被关闭时触发该事件。容器被关闭时，其管理的所有单例Bean都被销毁。 

-  RequestHandleEvent：在Web应用中，当一个http请求（request）结束触发该事件。 
- ContestStartedEvent：Spring2.5新增的事件，当容器调用ConfigurableApplicationContext的Start()方法开始/重新开始容器时触发该事件。 

- ContestStopedEvent：Spring2.5新增的事件，当容器调用ConfigurableApplicationContext的Stop()方法停止容器时触发该事件。 

```
@Component
public class ApplicationListenerImpl implements ApplicationListener<ContextRefreshedEvent> {

    private static final Logger logger = LoggerFactory.getLogger(ApplicationListenerImpl.class);

    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        /**
         * 系统两种容器：root application context 和项目名-servlet context ；
         * 下面代码防止执行两次
         */
        if(event.getApplicationContext().getParent() == null){

        }
    }
}
```