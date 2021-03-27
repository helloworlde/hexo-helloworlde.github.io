---
title: MySQL 中关于gap lock / next-key lock 的一个问题
date: 2019-01-07 21:40:58
tags:
    - MySQL
categories: 
    - MySQL
---

# MySQL 中关于gap lock / next-key lock 的一个问题

> 在学习 MySQL 的过程中遇到的一个关于锁的问题，包含多个 MySQL 相关的知识；相关资料在文章末尾

### 问题描述

- 表初始化

```sql
CREATE TABLE z (
  id INT PRIMARY KEY AUTO_INCREMENT,
  b  INT,
  KEY b(b)
)
  ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

INSERT INTO z (id, b)
VALUES (1, 2),
  (3, 4),
  (5, 6),
  (7, 8),
  (9, 10);
```

- session A

```sql
BEGIN;
SELECT *
FROM z
WHERE b = 6 FOR UPDATE;
```

- session B

```sql
INSERT INTO z VALUES (2, 4);/*success*/
INSERT INTO z VALUES (2, 8);/*blocked*/
INSERT INTO z VALUES (4, 4);/*blocked*/
INSERT INTO z VALUES (4, 8);/*blocked*/
INSERT INTO z VALUES (8, 4);/*blocked*/
INSERT INTO z VALUES (8, 8);/*success*/
INSERT INTO z VALUES (0, 4);/*blocked*/
INSERT INTO z VALUES (-1, 4);/*success*/
```

分别执行 session B中的insert 会出现上述情况，为什么？

### 加锁过程

1. 在索引 b 上的等值查询，给索引 b 加上了 next-key lock (4, 6]；索引向右遍历，且最后一个值不满足条件时退化为间隙锁；所以会再加上间隙锁 (6,8)；所以索引 b 上的 next-key lock 的范围是(b=4,id=3)到(b=6,id=5)这个左开右闭区间和(b=6,id=5)到(b=8,id=7)这个开区间
2. for update 会给 b = 6 这一行加上行锁；因此 (b=6,id=5) 这一行上有行锁

---

- 这么看来上述语句都不在锁的范围内，为什么会被锁

这个问题其实是因为没有理解索引的结构，所以认为所有值都不应该被锁

- 如图所示，此时索引 b 上的锁：

![索引 b 锁范围.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/mysql/lock/%E7%B4%A2%E5%BC%95%20b%20%E9%94%81%E8%8C%83%E5%9B%B4.png)

- 图中绿色部分即为被锁范围；索引会根据 b 和 id 的值进行排序，插入不同的值，锁的范围是不一样的；分别插入 (b=4,id=2) 和(b=4,id=4)，插入的位置如图所示：

![(2,4)&(4,4)索引 b 锁范围.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/mysql/lock/%282%2C4%29%26%284%2C4%29%E7%B4%A2%E5%BC%95%20b%20%E9%94%81%E8%8C%83%E5%9B%B4.png)

- 因为索引是有序的，此时，由于记录(b=4,id=3)的存在，(b=4,id=2)不在锁的范围内，可以插入，但(b=4,id=4)在锁的范围内，所以插入时需要等待锁释放，被 blocked
- 对于其他(id,b)的值(2,8),(4,8),(8,4),(8,8)也是同样的道理；因此，得出以下结论：
  - id <= 2 时，b 索引锁范围为 (4,8]
  - 2 < id < 8 时，b 索引锁范围为 [4,8]
  - a >= 8 时，b 索引锁范围为 [4,8)

![(2,8),(4,8),(8,4),(8,8)索引 b 锁范围.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/mysql/lock/%282%2C8%29%2C%284%2C8%29%2C%288%2C4%29%2C%288%2C8%29%E7%B4%A2%E5%BC%95%20b%20%E9%94%81%E8%8C%83%E5%9B%B4.png)

- 但是，实践发现，当 id=0 时，被锁的范围为 [4,8)，这和我们得到的第一个结论(4,8]不一样；此时分析得到的示意图为：

![(0,4),(0,8)索引 b 锁范围.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/mysql/lock/%280%2C4%29%2C%280%2C8%29%E7%B4%A2%E5%BC%95%20b%20%E9%94%81%E8%8C%83%E5%9B%B4.png)

- 在 session A 中尝试插入 (b=4, id=0)并查询结果：

```sql
INSERT INTO z
VALUES (0, 4);

SELECT *
FROM z;
```

- 此时，发现表中并没有发现 (b=4, id=0)这条记录，但是多了 (b=4,id=10)这条；所以插入 (b=4, id=0)的时候真正插入的是 (b=4,id=10)这个值；这是因为我们在创建表的时候指定主键 id 的值 `AUTO INCREMENT`，当插入的主键值为0的时候，会被替换为 `AUTO_INCREMENT`的值，即10

![(0,4)插入后查询结果.png](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/mysql/lock/%280%2C4%29%E6%8F%92%E5%85%A5%E5%90%8E%E6%9F%A5%E8%AF%A2%E7%BB%93%E6%9E%9C.png)

- 对此，MySQL 官方文档中的解释是：在非 `NO_AUTO_VALUE_ON_ZERO`模式下，给自增的列赋值为 0，都会被替换为自增序列的下一个值；当该自增列值指定 NOT NULL 时赋值 NULL，也会被替换；当插入其他值时，自增序列的值会被替换为当前列中最大值的下一个值；参考 [MySQL 8.0 Reference Manual  文档，Tutorial 的 Examples of Common Queries , 3.6.9 Using AUTO_INCREMENT](https://dev.mysql.com/doc/refman/8.0/en/example-auto-increment.html)

> No value was specified for the AUTO_INCREMENT column, so MySQL assigned sequence numbers automatically. You can also explicitly assign 0 to the column to generate sequence numbers, unless the NO_AUTO_VALUE_ON_ZERO SQL mode is enabled.
>
> If the column is declared NOT NULL, it is also possible to assign NULL to the column to generate sequence numbers.
>
> When you insert any other value into an AUTO_INCREMENT column, the column is set to that value and the sequence is reset so that the next automatically generated value follows sequentially from the largest column value.

- 如果将主键修改为不自增，插入 (b=4, id=0) 会得到这条记录

---

- 特别感谢 [@__丁奇](https://weibo.com/tdingqi?is_all=1) 老师的耐心解答，老师的专栏 [MySQL 实战 45讲](http://gk.link/a/101Si) 是一个非常好的专栏，内容和评论的质量都很高，想学习 MySQL 的同学可以订阅
- 文章相关的极客专栏为 [21 | 为什么我只改一行的语句，锁这么多？](https://time.geekbang.org/column/article/75659)

![专栏海报](https://hellowoodes.oss-cn-beijing.aliyuncs.com/blog/mysql/lock/MySQL%20%E4%B8%93%E6%A0%8F%E6%B5%B7%E6%8A%A5.jpeg)
