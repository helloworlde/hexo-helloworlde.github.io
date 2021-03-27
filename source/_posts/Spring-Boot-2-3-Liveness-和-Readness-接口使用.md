---
title: Spring Boot 2.3+ Liveness 和 Readness 接口使用
date: 2020-09-20 22:32:48
tags:
    - SpringBoot
categories: 
    - SpringBoot
---

# Spring Boot 2.3+ Liveness 和 Readness 接口使用

在 Spring Boot  2.3+ 中，提供了单独的 liveness 和 readness，用于为 Kubernetes 提供相应检查接口

- liveness
用于检查应用是否存活，当应用组件因故障不健康时，可以通过这个接口的结果，配置相应策略，重启应用或重新调度 Pod
- readness
用于检查应用是否就绪，是否可以提供服务，如当流量太大超过应用的承载范围时，可以将这个接口的状态改为不健康，这样可以停止接收流量，当处理完后再次检查时变为健康，继续处理请求

## 配置 

- build.gradle 添加依赖

```groovy
plugins {
    id 'org.springframework.boot' version '2.3.2.RELEASE'
    id 'io.spring.dependency-management' version '1.0.9.RELEASE'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

- application.properties 添加配置

从 2.3.2 开始，`/actuator/health` 接口添加了分组的概念，默认分为 liveness 和 readness 两个组，需要显式指定后才可以使用，否则会返回 404；`/actuator/health` 接口包含所有的指标

当前应用支持的指标，可以设置 `management.endpoint.health.show-details=always`后从 `/actuator/health` 接口获取

先指定 readness 指标，之后可以将剩余的所有指标设置为 liveness 的指标

```
management.endpoint.health.show-details=always
management.endpoint.health.group.readiness.include=ping
management.endpoint.health.group.liveness.include=*
management.endpoint.health.group.liveness.exclude=${management.endpoint.health.group.readiness.include}
```

## 测试 

- 启动应用

### 请求相应的接口

- health

```bash
curl http://localhost:8080/actuator/health
```

```json
{
    "status": "UP",
    "components": {
        "diskSpace": {
            "status": "UP",
            "details": {
                "total": 499963174912,
                "free": 380364800000,
                "threshold": 10485760,
                "exists": true
            }
        },
        "livenessStateProbeIndicator": {
            "status": "UP"
        },
        "ping": {
            "status": "UP"
        },
        "readinessStateProbeIndicator": {
            "status": "UP"
        }
    },
    "groups": [
        "liveness",
        "readiness"
    ]
}
```

- readiness

```bash
curl http://localhost:8080/actuator/health/readiness
```

```json
{
    "components": {
        "ping": {
            "status": "UP"
        }
    },
    "status": "UP"
}
```

- liveness

```bash
curl http://localhost:8080/actuator/health/liveness
```

```json
{
    "components": {
        "custom": {
            "details": {
                "Status": "Health"
            },
            "status": "UP"
        },
        "diskSpace": {
            "details": {
                "exists": true,
                "free": 380286464000,
                "threshold": 10485760,
                "total": 499963174912
            },
            "status": "UP"
        }
    },
    "status": "UP"
}
```

## 自定义检查项

和自定义健康检查项一样，添加一个新的 `HealthIndicator`，添加到相应的分组即可

- CustomHealthIndicator.java

```
@Component
public class CustomHealthIndicator implements HealthIndicator {

    private boolean health = true;

    @Override
    public Health health() {
        if (health) {
            return Health.up().withDetail("Status", "Health").build();
        }
        return Health.down().withDetail("Status", "Not Health").build();
    }

    public void setHealth(boolean health) {
        this.health = health;
    }
}
```

- applicaiton.properties

```
management.endpoint.health.group.readiness.include=ping,custom
```

### 测试

- readiness

```bash
curl http://localhost:8080/actuator/health/readiness
```

```json
{
    "components": {
        "custom": {
            "details": {
                "Status": "Health"
            },
            "status": "UP"
        },
        "ping": {
            "status": "UP"
        }
    },
    "status": "UP"
}
```

