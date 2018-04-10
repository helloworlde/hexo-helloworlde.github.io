---
title: Rocket MQ 相关知识
date: 2018-01-01 12:54:02
tags:
    - Java
    - Rocket MQ 
categories: 
    - Java
    - Rocket MQ
---
# RocketMQ 相关知识

@(消息队列)[RocketMQ, 消息]

> Rocket MQ消息队列（Message Queue，简称 MQ）是阿里巴巴集团中间件技术部自主研发的专业消息中间件。产品基于高可用分布式集群技术，提供消息发布订阅、消息轨迹查询、定时（延时）消息、资源统计、监控报警等一系列消息云服务，是企业级互联网架构的核心产品。

## Rocket MQ相关名词

- Producer 消息生产者，负责生产消息
- Consumer 消息消费者，负责消费消息
- NameServer 无状态节点，用来保存活跃的broker列表和topic列表
- Broker 消息中转角色，负责存储消息，转发消息
- Topic 消息的逻辑管理单位
- Message 消息
    - body 消息体，用于携带消息具体内容
    - key 消息的key，用于区别不同的消息
    - tags 消息的Tag，用于不同的订阅者过滤消息

## 消息发送方式
- 同步方式
> 发送消息，接收到结果之后再发送下一条消息，速度最慢，耗时最长
- 异步方式
> 发送消息，不论是否收到结果，直接发送下一条消息，发送速度介于同步和单向方式之间
- 单向方式 
> 发送消息，直接发送消息，不返回发送结果，发送速度最快

## 消息类型
- 定时消息
> 在指定的发送时间发送消息
- 延时消息
> 从当前时间开始，经过延时时间后再发送消息
- 顺序消息
> 立即发送消息
- 事务消息
> MQ 提供类似 X/Open XA 的分布事务功能，通过 MQ 事务消息能达到分布式事务的最终一致

# 实例代码
## Producer
```
public class ProducerDelayTest {
    public static void main(String[] args) {
        Properties properties = new Properties();
        //您在 MQ 控制台创建的Producer ID
        properties.put(PropertyKeyConst.ProducerId, "XXX");
        // 阿里云身份验证，在阿里云服务器管理控制台创建
        properties.put(PropertyKeyConst.AccessKey, "XXX");
        // 阿里云身份验证，在阿里云服务器管理控制台创建
        properties.put(PropertyKeyConst.SecretKey, "XXX");
        // 设置 TCP 接入域名（此处以公共云生产环境为例）
        properties.put(PropertyKeyConst.ONSAddr,
          "http://onsaddr-internal.aliyun.com:8080/rocketmq/nsaddr4client-internal");
        Producer producer = ONSFactory.createProducer(properties);
        // 在发送消息前，必须调用start方法来启动Producer，只需调用一次即可。
        producer.start();
      
            /**
              *  消息类型代码，参考下面消息类型代码        
              */
             /**
               *  消息发送方式代码，参考下面发送方式代码        
               */
        System.out.println("Message Id:" + sendResult.getMessageId());
        // 在应用退出前，销毁 Producer 对象
        // 注意：如果不销毁也没有问题，如果发送消息较多不应该销毁
        producer.shutdown();
    }
}
```

### 消息类型代码
- 定时消息
``` 
        Message msg = new Message();
        msg.setTag("TAG");
        msg.setKey("KEY");
        msg.setTopic("TOPIC");
        msg.setBody("BODY".getBytes());
        long timeStamp =new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse("2017-09-03 16:21:00").getTime();
        msg.setStartDeliverTime(timeStamp);
       
```
- 延时消息
```
        Message msg = new Message();
        msg.setTag("TAG");
        msg.setKey("KEY");
        msg.setTopic("TOPIC");
        msg.setBody("BODY".getBytes()); 
        long delayTime = 3000;//30秒后再发送
        msg.setStartDeliverTime(System.currentTimes() + delayTime);
```

- 顺序消息
```
        Message msg = new Message();
        msg.setTag("TAG");
        msg.setKey("KEY");
        msg.setTopic("TOPIC".getBytes());
        msg.setBody("BODY"); 
```

### 消息发送方式代码
- 同步方式发送
```
        SendResult sendResult = producer.send(msg);
```

- 异步方式发送
```
        producer.sendAsync(message, new SendCallback() {
            @Override
            public void onSuccess(final SendResult sendResult) {
                logger.info("MQ send ASYNCHRONOUS message successed，response is " + JSON.toJSONString(sendResult));
            }

            @Override
            public void onException(OnExceptionContext onExceptionContext) {
                logger.info("MQ send ASYNCHRONOUS message failed, error is " + onExceptionContext.getException().getMessage());
            }
        });
```
- 单向方式发送
```
         producer.sendOneway(message);
```

## Consumer
```
public class ConsumerTest {
    public static void main(String[] args) {
        Properties properties = new Properties();
        // 您在控制台创建的 Consumer ID
        properties.put(PropertyKeyConst.ConsumerId, "XXX");
        // AccessKey 阿里云身份验证，在阿里云服务器管理控制台创建
        properties.put(PropertyKeyConst.AccessKey, "XXX");
        // SecretKey 阿里云身份验证，在阿里云服务器管理控制台创建
        properties.put(PropertyKeyConst.SecretKey, "XXX");
        // 设置 TCP 接入域名（此处以公共云生产环境为例）
        properties.put(PropertyKeyConst.ONSAddr,
          "http://onsaddr-internal.aliyun.com:8080/rocketmq/nsaddr4client-internal");
          // 集群订阅方式 (默认)
          // properties.put(PropertyKeyConst.MessageModel, PropertyValueConst.CLUSTERING);
          // 广播订阅方式
          // properties.put(PropertyKeyConst.MessageModel, PropertyValueConst.BROADCASTING);
        Consumer consumer = ONSFactory.createConsumer(properties);
        consumer.subscribe("TopicTestMQ", "TagA||TagB", new MessageListener() { //订阅多个Tag
            public Action consume(Message message, ConsumeContext context) {
                System.out.println("Receive: " + message);
                return Action.CommitMessage;
            }
        });
        //订阅另外一个Topic
        consumer.subscribe("TopicTestMQ-Other", "*", new MessageListener() { //订阅全部Tag
            public Action consume(Message message, ConsumeContext context) {
                System.out.println("Receive: " + message);
                return Action.CommitMessage;
            }
        });
        consumer.start();
        System.out.println("Consumer Started");
    }
}
```