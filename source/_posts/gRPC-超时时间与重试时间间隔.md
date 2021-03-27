---
title: gRPC 超时时间与重试时间间隔
date: 2020-09-20 22:41:50
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 超时时间与重试时间间隔

> gRPC 的超时时间生效机制以及重试超时时间间隔 

## 超时时间配置

对指定的服务和方法单独设置超时时间，timeout 作用于所有的 RPC 请求（无论是否重试，都会在 timeout 的时间之后超时，进行中的重试请求会被取消）

```json
{
    "methodConfig": [
        {
            "name": [
                {
                    "service": "io.github.helloworlde.grpc.UserInfoService"
                }
            ],
            "retryPolicy": {
                "maxAttempts": 5,
                "initialBackoff": "0.01s",
                "maxBackoff": "0.1s",
                "backoffMultiplier": 2,
                "retryableStatusCodes": [
                    "UNAVAILABLE"
                ]
            },
            "waitForReady": false,
            "timeout": "3s"
        }
    ]
}
```

## 重试时间间隔

gRPC 的重试时间间隔是由 `initialBackoff`, `maxBackoff`, `backoffMultiplier` 共同决定的，实现方法是 `io.grpc.internal.RetriableStream.Sublistener#makeRetryDecision`

```java
if (retryPolicy.maxAttempts > substream.previousAttemptCount + 1 && !isThrottled) {
  if (pushbackMillis == null) {
    if (isRetryableStatusCode) {
      shouldRetry = true;
      backoffNanos = (long) (nextBackoffIntervalNanos * random.nextDouble());
      nextBackoffIntervalNanos = Math.min((long) (nextBackoffIntervalNanos * retryPolicy.backoffMultiplier), retryPolicy.maxBackoffNanos);
    }
  } else if (pushbackMillis >= 0) {
    shouldRetry = true;
    backoffNanos = TimeUnit.MILLISECONDS.toNanos(pushbackMillis);
    nextBackoffIntervalNanos = retryPolicy.initialBackoffNanos;
  }
}
```

如果开启了重试，且没有达到节流的限制，如果服务端有回推延时时间，则使用服务端回推的时间作为延时时间，初始延迟时间作为下次延时的初始时间；

如果服务端没有回推延时时间，第一次重试使用初始时间乘以随机数，作为延时时间；之后的延时时间使用上一次初始延时时间乘以退避指数 backoffMultiplier，与最大延迟时间取最小值，再乘以随机数作为延迟时间

如果 初始延迟时间(initialBackoff)是 1s，最大延迟时间(maxBackoff) 是 10s，退避指数(backoffMultiplier) 是2：
- 第一次延时延时: `d1 = 1s * random.nextDouble() < 1s`
- 第二次重试延时: `d2 = Math.min(1s * 2, 10s) * random.nextDouble() = 2s * random.nextDouble()  < 2s`
- 第三次重试延时: `d3 = Math.min(2s * 2, 10s) * random.nextDouble() = 4s * random.nextDouble()  < 4s`
- 第四次重试延时: `d4 = Math.min(4s * 2, 10s) * random.nextDouble() = 8s * random.nextDouble()  < 8s`
所有重试延时：1s + 2s + 4s + 8s < 15s，所以此时的 timeout 合理的值应该为 15s