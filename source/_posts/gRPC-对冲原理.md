---
title: gRPC 对冲原理
date: 2020-09-20 22:39:26
tags:
    - gRPC
categories: 
    - gRPC
---

# gRPC 对冲原理

gRPC 对冲开启后，当请求在指定的时间间隔后没有返回时，会发起对冲请求，继续等待，如果依然没有返回，则重复发送直到接收到返回结果或者超时取消

对冲适用于当下游服务部分节点故障无法及时响应或者响应不及时的场景，通过对冲可以减少请求的失败率，但是可能会导致延时增加

对冲和重试的流程相似，在第一次发起请求的时候根据服务名和方法名决定使用哪种策略；如果是对冲策略，则在发起请求时提交一个延时任务，这个任务会发起一个新的请求，并在执行的时候再发起一个请求，并将这些请求添加到队列中；多个请求哪个先返回就使用哪个请求的结果，将其他的请求取消并提交流


## 执行流程

- io.grpc.internal.RetriableStream#start 

开始第一个 RPC 调用

```
  @Override
  public final void start(ClientStreamListener listener) {
    // 构造一个 BufferEntry
    class StartEntry implements BufferEntry {
      @Override
      public void runWith(Substream substream) {
        substream.stream.start(new Sublistener(substream));
      }
    }

    synchronized (lock) {
      // 新建 BufferEntry，添加到 buffer 中
      state.buffer.add(new StartEntry());
    }

    // 创建 Substream
    Substream substream = createSubstream(0);
    hedgingPolicy = hedgingPolicyProvider.get();
    // 如果有对冲策略
    if (!HedgingPolicy.DEFAULT.equals(hedgingPolicy)) {
      // 如果对冲策略有效，则将重试策略置为 null
      isHedging = true;
      retryPolicy = RetryPolicy.DEFAULT;

      FutureCanceller scheduledHedgingRef = null;

      synchronized (lock) {
        // 将这个流添加到对冲中
        state = state.addActiveHedge(substream);
        // 没有提交的流，且没有达到最大对冲次数，且没有终止，且没有节流或没有达到节流阈值
        // 则创建对冲 Future
        if (hasPotentialHedging(state) && (throttle == null || throttle.isAboveThreshold())) {
          scheduledHedging = scheduledHedgingRef = new FutureCanceller(lock);
        }
      }

      // 如果对冲请求不为空，则提交延时任务
      if (scheduledHedgingRef != null) {
        scheduledHedgingRef.setFuture(
                scheduledExecutorService.schedule(
                        new HedgingRunnable(scheduledHedgingRef),
                        hedgingPolicy.hedgingDelayNanos,
                        TimeUnit.NANOSECONDS)
        );
      }
    }
    // 消耗缓冲的请求
    drain(substream);
  }
```

- io.grpc.internal.RetriableStream.HedgingRunnable

对冲任务，在执行的时候发起请求，并根据当前的提交的请求数量、状态等判断是否需要取消，如果不取消则再次提交一个延时任务

```java
  private final class HedgingRunnable implements Runnable {
    @Override
    public void run() {
      callExecutor.execute(new Runnable() {
        @Override
        public void run() {
          // 创建对冲流
          Substream newSubstream = createSubstream(state.hedgingAttemptCount);
          boolean cancelled = false;
          FutureCanceller future = null;

          synchronized (lock) {
            // 如果请求被取消了，则取消流
            if (scheduledHedgingRef.isCancelled()) {
              cancelled = true;
            } else {
              // 将新创建的流添加到对冲流集合中
              state = state.addActiveHedge(newSubstream);
              // 没有提交的流，且没有达到最大对冲次数，且没有终止，且没有节流或者没有达到节流阈值，
              // 则创建 Future
              if (hasPotentialHedging(state) && (throttle == null || throttle.isAboveThreshold())) {
                scheduledHedging = future = new FutureCanceller(lock);
              } else {
                // 否则冻结对冲
                state = state.freezeHedging();
                scheduledHedging = null;
              }
            }
          }

          // 如果请求已经取消了，则取消流并返回
          if (cancelled) {
            newSubstream.stream.cancel(Status.CANCELLED.withDescription("Unneeded hedging"));
            return;
          }
          // 如果对冲请求不为空，则提交延时任务
          if (future != null) {
            future.setFuture(
                    scheduledExecutorService.schedule(
                            new HedgingRunnable(future),
                            hedgingPolicy.hedgingDelayNanos,
                            TimeUnit.NANOSECONDS));
          }
          // 消耗缓冲的请求
          drain(newSubstream);
        }
      });
    }
  }
```

