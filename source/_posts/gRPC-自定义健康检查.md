---
title: gRPC  自定义健康检查
date: 2020-09-20 22:38:15
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC  自定义健康检查

在 gRPC 中自定义健康检查逻辑，用于检查特定的组件(如检查 Redis、MQ 等)，或者结合其他的服务组件一起使用(如使用 Spring Boot 的健康检查)

## 实现

gRPC 的健康检查服务是通过 `health.proto`定义的

- health.proto

```protobuf
syntax = "proto3";

package grpc.health.v1;

option csharp_namespace = "Grpc.Health.V1";
option go_package = "google.golang.org/grpc/health/grpc_health_v1";
option java_multiple_files = true;
option java_outer_classname = "HealthProto";
option java_package = "io.grpc.health.v1";

message HealthCheckRequest {
  string service = 1;
}
message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}

service Health {
  // 单次健康检查
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);

  // 流式健康检查
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

里面定义了两个方法，一个是用于单次检查的 `Check`方法，一个是用于流式请求的 `Watch`方法

### 自定义检查组件

- CustomHealthCheckImpl.java

自定义健康检查逻辑，通过不同的组件名称返回相应的状态信息

```java
public class CustomHealthCheckImpl extends HealthGrpc.HealthImplBase {
    @Override
    public void check(HealthCheckRequest request, StreamObserver<HealthCheckResponse> responseObserver) {
        System.out.println("健康检查:" + request.getService());

        HealthCheckResponse.ServingStatus servingStatus = getServingStatus(request);

        HealthCheckResponse response = HealthCheckResponse.newBuilder()
                                                          .setStatus(servingStatus)
                                                          .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void watch(HealthCheckRequest request, StreamObserver<HealthCheckResponse> responseObserver) {
        System.out.println("健康检查 Stream:" + request.getService());

        HealthCheckResponse.ServingStatus servingStatus = getServingStatus(request);

        HealthCheckResponse response = HealthCheckResponse.newBuilder()
                                                          .setStatus(servingStatus)
                                                          .build();
        responseObserver.onNext(response);
    }

    private HealthCheckResponse.ServingStatus getServingStatus(HealthCheckRequest request) {
        HealthCheckResponse.ServingStatus servingStatus;
        String service = request.getService();

        switch (service) {
            case "":
                servingStatus = HealthCheckResponse.ServingStatus.SERVING;
                break;
            case "mysql":
                servingStatus = checkMySQL();
                break;
            case "redis":
                servingStatus = checkRedis();
                break;
            default:
                servingStatus = HealthCheckResponse.ServingStatus.UNKNOWN;
                break;
        }
        return servingStatus;
    }
}
```

### 使用 

在 Server 端添加自定义的健康检查服务

```java
Server server = ServerBuilder.forPort(1234)
                             .addService(new UserInfoServiceImpl()) 
                             .addService(new HelloServiceImpl())
                             .addService(new CustomHealthCheckImpl())
                             .build();
```

### 测试 

```bash
grpc-health-probe -addr localhost:1234 -service mysql
status: SERVING

grpc-health-probe -addr localhost:1234 -service redis
service unhealthy (responded with "NOT_SERVING")
```

### 使用 Spring Boot 健康检查

通过 Spring Boot  的 HealthIndicator 作为 gRPC 的健康检查，需要将相应的组件状态转为 gRPC 的健康检查状态

```java
public class CustomHealthCheckImpl extends HealthGrpc.HealthImplBase {

    private final ObjectProvider<HealthIndicator> healthIndicatorObjectProvider;

    public CustomHealthCheckImpl(ObjectProvider<HealthIndicator> healthIndicatorObjectProvider) {
        this.healthIndicatorObjectProvider = healthIndicatorObjectProvider;
    }

    @Override
    public void check(HealthCheckRequest request, StreamObserver<HealthCheckResponse> responseObserver) {
        HealthCheckResponse.ServingStatus servingStatus = getServingStatus(request);

        HealthCheckResponse response = HealthCheckResponse.newBuilder()
                                                          .setStatus(servingStatus)
                                                          .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void watch(HealthCheckRequest request, StreamObserver<HealthCheckResponse> responseObserver) {
        HealthCheckResponse.ServingStatus servingStatus = getServingStatus(request);

        HealthCheckResponse response = HealthCheckResponse.newBuilder()
                                                          .setStatus(servingStatus)
                                                          .build();
        responseObserver.onNext(response);
    }

    private HealthCheckResponse.ServingStatus getServingStatus(HealthCheckRequest request) {
        HealthCheckResponse.ServingStatus servingStatus;
        String service = request.getService();

        servingStatus = healthIndicatorObjectProvider.stream()
                                                     .peek(h -> {
                                                         System.out.println(h.getClass().getSimpleName());
                                                     })
                                                     .filter(h -> h.getClass().getSimpleName().toLowerCase().contains(service.toLowerCase()))
                                                     .map(HealthIndicator::health)
                                                     .map(Health::getStatus)
                                                     .map(this::toGrpcHealthStatus)
                                                     .findFirst()
                                                     .orElse(HealthCheckResponse.ServingStatus.UNKNOWN);

        return servingStatus;
    }

    private HealthCheckResponse.ServingStatus toGrpcHealthStatus(Status status) {
        HealthCheckResponse.ServingStatus servingStatus;

        if (Status.UP.equals(status)) {
            servingStatus = HealthCheckResponse.ServingStatus.SERVING;
        } else if (Status.DOWN.equals(status)) {
            servingStatus = HealthCheckResponse.ServingStatus.NOT_SERVING;
        } else if (Status.OUT_OF_SERVICE.equals(status)) {
            servingStatus = HealthCheckResponse.ServingStatus.SERVICE_UNKNOWN;
        } else {
            servingStatus = HealthCheckResponse.ServingStatus.UNKNOWN;
        }
        return servingStatus;
    }

}
```

- 测试

```bash
grpc-health-probe -addr localhost:9090 -service disk
status: SERVING

grpc-health-probe -addr localhost:9090 -service redis
service unhealthy (responded with "UNKNOWN")
```