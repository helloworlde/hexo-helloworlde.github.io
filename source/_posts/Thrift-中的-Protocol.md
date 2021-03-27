---
title: Thrift 中的 Protocol
date: 2021-01-31 22:34:46
tags:
    - Thrift
categories: 
    - Thrift
---

# Thrift 中的 Protocol

`TProtocol` 是 Thrift 中协议的抽象类，定义了数据序列化和反序列化的接口

![thrift-java-source-class-protocol.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/picture/thrift-java-source-class-protocol.png)

## 属性

`TProtocol` 中有 `TTransport`类型的属性`trans_`，用于与底层的传输层进行数据交互

## 方法

`TProtocol` 中的方法可以分为两类，分别用于写入和读取各种类型
其中 `Message`，`Struct`, `Field`,`Map`,`List`,`Set` 等类型会有开始和结束标志，一些还会写入或读取名称、序号等信息；可以参考 [Thrift Protocol Structure](https://github.com/helloworlde/thrift/blob/master/doc/specs/thrift-protocol-spec.md)

```java
/**
 * 写入
 */
public abstract void writeMessageBegin(TMessage message) throws TException;
public abstract void writeMessageEnd() throws TException;
public abstract void writeStructBegin(TStruct struct) throws TException;
public abstract void writeStructEnd() throws TException;
public abstract void writeFieldBegin(TField field) throws TException;
public abstract void writeFieldEnd() throws TException;
public abstract void writeFieldStop() throws TException;
public abstract void writeMapBegin(TMap map) throws TException;
public abstract void writeMapEnd() throws TException;
public abstract void writeListBegin(TList list) throws TException;
public abstract void writeListEnd() throws TException;
public abstract void writeSetBegin(TSet set) throws TException;
public abstract void writeSetEnd() throws TException;
public abstract void writeBool(boolean b) throws TException;
public abstract void writeByte(byte b) throws TException;
public abstract void writeI16(short i16) throws TException;
public abstract void writeI32(int i32) throws TException;
public abstract void writeI64(long i64) throws TException;
public abstract void writeDouble(double dub) throws TException;
public abstract void writeString(String str) throws TException;
public abstract void writeBinary(ByteBuffer buf) throws TException;
/**
 * 读取
 */
public abstract TMessage readMessageBegin() throws TException;
public abstract void readMessageEnd() throws TException;
public abstract TStruct readStructBegin() throws TException;
public abstract void readStructEnd() throws TException;
public abstract TField readFieldBegin() throws TException;
public abstract void readFieldEnd() throws TException;
public abstract TMap readMapBegin() throws TException;
public abstract void readMapEnd() throws TException;
public abstract TList readListBegin() throws TException;
public abstract void readListEnd() throws TException;
public abstract TSet readSetBegin() throws TException;
public abstract void readSetEnd() throws TException;
public abstract boolean readBool() throws TException;
public abstract byte readByte() throws TException;
public abstract short readI16() throws TException;
public abstract int readI32() throws TException;
public abstract long readI64() throws TException;
public abstract double readDouble() throws TException;
public abstract String readString() throws TException;
public abstract ByteBuffer readBinary() throws TException;
```

## 实现类

- `TBinaryProtocol`: 二进制协议，根据 Thrift 的类型按  [Thrift Protocol Structure](https://github.com/helloworlde/thrift/blob/master/doc/specs/thrift-protocol-spec.md) 定义写入数据；参考 [Thrift Binary protocol encoding](https://github.com/helloworlde/thrift/blob/master/doc/specs/thrift-binary-protocol.md)

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

- `TCompactProtocol`：压缩协议，会将请求内容进行压缩后写入，参考 [Thrift Compact protocol encoding
](https://github.com/helloworlde/thrift/blob/master/doc/specs/thrift-compact-protocol.md)

```java
public void writeMessageBegin(TMessage message) throws TException {
    writeByteDirect(PROTOCOL_ID);
    writeByteDirect((VERSION & VERSION_MASK) | ((message.type << TYPE_SHIFT_AMOUNT) & TYPE_MASK));
    writeVarint32(message.seqid);
    writeString(message.name);
}
```

- `TTupleProtocol`：继承了 `TCompactProtocol` 类，Scheme 使用 `TupleScheme`，表示使用写消息体的方式序列化和反序列化，而不是 `StandardScheme` 使用消息头和消息体的方式序列化和反序列化

```java
public void writeBitSet(BitSet bs, int vectorWidth) throws TException {
    byte[] bytes = toByteArray(bs, vectorWidth);
    for (byte b : bytes) {
        writeByte(b);
    }
}
```

- `TJSONProtocol`：将消息序列化为 JSON，可以用于泛化调用的场景下

```java
public void writeMessageBegin(TMessage message) throws TException {
    resetContext(); // THRIFT-3743
    writeJSONArrayStart();
    writeJSONInteger(VERSION);
    byte[] b = message.name.getBytes(StandardCharsets.UTF_8);
    writeJSONString(b);
    writeJSONInteger(message.type);
    writeJSONInteger(message.seqid);
}
```

- `TSimpleJSONProtocol`: 将消息以 JSON 格式输出，没有实现读取，用于脚本语言

```java
public void writeMessageBegin(TMessage message) throws TException {
    resetWriteContext(); // THRIFT-3743
    trans_.write(LBRACKET);
    pushWriteContext(new ListContext());
    writeString(message.name);
    writeByte(message.type);
    writeI32(message.seqid);
}
```

- `TProtocolDecorator`：抽象实现，会将所有的操作都转发给被代理的类实现

```java
public void writeMessageBegin(TMessage tMessage) throws TException {
    concreteProtocol.writeMessageBegin(tMessage);
}
```

- `TMultiplexedProtocol`：`TProtocolDecorator` 的实现类，在消息头部写入了服务的名称，会被 Server 端解析；用于有多个服务的 Server；其他的类型写入和读取由被代理的协议实现

```java 
public void writeMessageBegin(TMessage tMessage) throws TException {
    if (tMessage.type == TMessageType.CALL || tMessage.type == TMessageType.ONEWAY) {
        super.writeMessageBegin(new TMessage(
                SERVICE_NAME + SEPARATOR + tMessage.name,
                tMessage.type,
                tMessage.seqid
        ));
    } else {
        super.writeMessageBegin(tMessage);
    }
}
```
- `StoredMessageProtocol`：`TProtocolDecorator` 的实现类，代理其他协议，通常用于 Server 端，只获取请求头，具体的读取由被代理的协议实现

```java
public TMessage readMessageBegin() throws TException {
    return messageBegin;
}
```