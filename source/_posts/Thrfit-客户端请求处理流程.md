---
title: Thrfit 客户端请求处理流程
date: 2021-02-20 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrfit 客户端请求处理流程

使用同步的非阻塞的服务端和客户端的请求处理流程

## 实现 

### IDL

- helloworld.thrift

```thrift
namespace java io.github.helloworlde.thrift

struct HelloMessage {
    1: required string message,
}

struct HelloResponse {
    1: required string message,
}

service HelloService {
    HelloResponse sayHello(1: HelloMessage request);
}
```

### 客户端实现

使用 `TSocket` 作为底层连接，协议使用 `TBinaryProtocol`

```java
try {
    TTransport transport  = new TSocket("localhost", 9090);
    transport.open();
    TProtocol protocol = new TBinaryProtocol(transport);

    HelloService.Client client = new HelloService.Client(protocol);

    HelloMessage request = new HelloMessage();
    request.setMessage("Thrift");

    HelloResponse response = client.sayHello(request);
    log.info("返回响应: {}", response.getMessage());

} catch (TException e) {
    e.printStackTrace();
}
```

## 请求处理流程

### 1. 建立连接

```java
transport.open();
```

- org.apache.thrift.transport.TSocket#open

初始化 Socket，建立连接

```java
public void open() throws TTransportException {
    if (socket_ == null) {
        // 初始化 Socket
        initSocket();
    }

    try {
        // 建立连接
        socket_.connect(new InetSocketAddress(host_, port_), connectTimeout_);
        // 初始化流
        inputStream_ = new BufferedInputStream(socket_.getInputStream());
        outputStream_ = new BufferedOutputStream(socket_.getOutputStream());
    } catch (IOException iox) {
        close();
        throw new TTransportException(TTransportException.NOT_OPEN, iox);
    }
}
```

###  2. 执行请求

使用 `TProtocol` 构建 `TServiceClient`，用于发送同步请求

- io.github.helloworlde.thrift.HelloService.Client#sayHello

```java
public HelloResponse sayHello(HelloMessage request) throws org.apache.thrift.TException {
    send_sayHello(request);
    return recv_sayHello();
}
```

#### 发送请求

- io.github.helloworlde.thrift.HelloService.Client#send_sayHello

其中的 `sayHello_args` 用于读写结构体，将消息内容转换为相应格式的字节

```java
public void send_sayHello(HelloMessage request) throws org.apache.thrift.TException {
    sayHello_args args = new sayHello_args();
    args.setRequest(request);
    sendBase("sayHello", args);
}
```

- org.apache.thrift.TServiceClient#sendBase(java.lang.String, org.apache.thrift.TBase<?,?>)

这里设置了消息类型是调用

```java
protected void sendBase(String methodName, TBase<?, ?> args) throws TException {
    sendBase(methodName, args, TMessageType.CALL);
}
```

- org.apache.thrift.TServiceClient#sendBase(java.lang.String, org.apache.thrift.TBase<?,?>, byte)

在写入请求，写入请求头，写入消息体，然后写入结尾符，将请求发送出去

```java
private void sendBase(String methodName, TBase<?, ?> args, byte type) throws TException {
    // 构建请求，写入头信息
    oprot_.writeMessageBegin(new TMessage(methodName, type, ++seqid_));
    // 写入协议对象
    args.write(oprot_);
    // 写入结尾信息
    oprot_.writeMessageEnd();
    // 清空缓冲，写入
    oprot_.getTransport().flush();
}
```

- org.apache.thrift.protocol.TBinaryProtocol#writeMessageBegin

会将版本、调用类型、方法名、请求 ID 写入到请求头

```java
public void writeMessageBegin(TMessage message) throws TException {
    if (strictWrite_) {
        // 写入版本
        int version = VERSION_1 | message.type;
        writeI32(version);
        // 被调用方法
        writeString(message.name);
        // 请求序号
        writeI32(message.seqid);
    } else {
        writeString(message.name);
        writeByte(message.type);
        writeI32(message.seqid);
    }
}
```

- io.github.helloworlde.thrift.HelloService.sayHello_args.sayHello_argsStandardScheme#write

写入请求内容，会将结构体的相关描述信息写入到请求中

```java
public void write(org.apache.thrift.protocol.TProtocol oprot, sayHello_args struct) throws org.apache.thrift.TException {
  struct.validate();

  oprot.writeStructBegin(STRUCT_DESC);
  if (struct.request != null) {
    oprot.writeFieldBegin(REQUEST_FIELD_DESC);
    struct.request.write(oprot);
    oprot.writeFieldEnd();
  }
  oprot.writeFieldStop();
  oprot.writeStructEnd();
}
```

#### 接收响应

- io.github.helloworlde.thrift.HelloService.Client#recv_sayHello

在处理请求时先构建了 `sayHello_result`对象，用于解析响应的描述

```java
public HelloResponse recv_sayHello() throws org.apache.thrift.TException {
  sayHello_result result = new sayHello_result();
  receiveBase(result, "sayHello");
  if (result.isSetSuccess()) {
    return result.success;
  }
  throw new org.apache.thrift.TApplicationException(org.apache.thrift.TApplicationException.MISSING_RESULT, "sayHello failed: unknown result");
}
```

- org.apache.thrift.TServiceClient#receiveBase

读取响应内容，解析为对象

```java
protected void receiveBase(TBase<?, ?> result, String methodName) throws TException {
    // 读取消息
    TMessage msg = iprot_.readMessageBegin();
    // 如果是异常，则读取异常并抛出
    if (msg.type == TMessageType.EXCEPTION) {
        TApplicationException x = new TApplicationException();
        x.read(iprot_);
        iprot_.readMessageEnd();
        throw x;
    }
    // 如果请求序号不匹配，则抛出异常
    if (msg.seqid != seqid_) {
        throw new TApplicationException(TApplicationException.BAD_SEQUENCE_ID,
                String.format("%s failed: out of sequence response: expected %d but got %d", methodName, seqid_, msg.seqid));
    }
    // 读取响应内容
    result.read(iprot_);
    iprot_.readMessageEnd();
}
```

- org.apache.thrift.protocol.TBinaryProtocol#readMessageBegin

读取并校验版本，获取方法名称、消息类型、请求 ID 

```java
public TMessage readMessageBegin() throws TException {
    int size = readI32();
    if (size < 0) {
        int version = size & VERSION_MASK;
        if (version != VERSION_1) {
            throw new TProtocolException(TProtocolException.BAD_VERSION, "Bad version in readMessageBegin");
        }
        return new TMessage(readString(), (byte) (size & 0x000000ff), readI32());
    } else {
        if (strictRead_) {
            throw new TProtocolException(TProtocolException.BAD_VERSION, "Missing version in readMessageBegin, old client?");
        }
        return new TMessage(readStringBody(size), readByte(), readI32());
    }
}
```

- io.github.helloworlde.thrift.HelloService.sayHello_result.sayHello_resultStandardScheme#read

读取响应内容，解析为相应的对象，然后赋值给 result 对象

```java
public void read(org.apache.thrift.protocol.TProtocol iprot, sayHello_result struct) throws org.apache.thrift.TException {
  org.apache.thrift.protocol.TField schemeField;
  iprot.readStructBegin();
  while (true) {
    schemeField = iprot.readFieldBegin();
    if (schemeField.type == org.apache.thrift.protocol.TType.STOP) {
      break;
    }
    switch (schemeField.id) {
      case 0: // SUCCESS
        if (schemeField.type == org.apache.thrift.protocol.TType.STRUCT) {
          struct.success = new HelloResponse();
          struct.success.read(iprot);
          struct.setSuccessIsSet(true);
        } else {
          org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
        }
        break;
      default:
        org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
    }
    iprot.readFieldEnd();
  }
  iprot.readStructEnd();

  // check for required fields of primitive type, which can't be checked in the validate method
  struct.validate();
}
```

## 参考文档

- [helloworlde/thrift-java-sample](https://github.com/helloworlde/thrift-java-sample)