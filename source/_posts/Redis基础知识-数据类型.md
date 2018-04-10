---
title: Redis基础知识-数据类型
date: 2018-01-01 12:07:03
tags:
    - Redis
categories: 
    - Redis
---

> Redis支持5种数据类型：字符串（string），哈希（hash），列表（list），集合（set），有序集合（sorted set）

## 字符串（string）
> string 是 Redis最基本的类型，一个key对应一个value，string可以包含任何数据，比如jpg图片或者序列化的对象，string是Redis最基本的类型，一个键最大能存储512MB

```
保存：SET key value
读取：get key 
```
## 哈希（Hash）
> Hash 是一个键名对集合，Hash是一个string类型的field和value的映射表，Hash特备适合存储对象

```
保存：hmset key value1 value2 value3
读取：hgetall key
```

## 列表（List）
> 列表是简单的字符串列表，按照插入顺序排序，可以将元素添加到头部（左边）或尾部（右边）

```
保存：
    lpush key value1 value2
    lpush key value3
读取：lrange key 0 10
```

## 集合（Set）
> Redis 的set是string类型的无序集合，集合是通过哈希表实现，添加、删除、查找的复杂度都是O(1)

```
保存：
    sadd key value1
    sadd key value2
    sadd key value3
读取：smembers key
```

## 有序集合（Sort Set）
> Redis ZSet和set一样也是string类型元素的集合，且不允许重复的成员，不同的是每个元素都会关联一个double类型的分数，Redis通过这个分数来为集合中的成员进行从小到大的排序，zset的成员是唯一的，但分数（score）可以重复

```:
保存：
    zadd key score1 value1 
    zadd key score2 value2 
    zadd key score3 value3
读取：zrangebyscore key 0 10
```