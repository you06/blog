title: TiDB 事务原理 - 点查询
author: you06
date: 2020-07-29 07:39:45
tags:
---
近来在阅读`TiDB`事务方面的代码，所以写一些学习记录。

点查询指的是获取单条数据的`SQL`语句，相比于范围查询，点查询的执行过程要简单很多，原因有以下几点：

* 点查询的结果只可能在一个 region 内
* 点查询只会涉及到一个 key 上的锁处理

以`TiDB`测试用例中的点查询来举例：

```SQL
use test;
create table point (id int primary key, c int, d varchar(10), unique c_d (c, d));
insert point values (1, 1, 'a');
insert point values (2, 2, 'b');

select * from point where id = 1 and c = 0;
select * from point where id < 0 and c = 1 and d = 'b';
select id as ident from point where id = 1;
```

在完成建表和插入数据后，我们进行了三个查询，其中第一条`SQL`可以通过在主键`id`的等于条件查询条件知道最多有一条结果；第二条`SQL`可以通过`c`和`d`的等于查询条件结合`unique c_d (c, d)`的约束知道最多有一条结果；第三条`SQL`和第一条一样。然而在实际执行时，还需要考虑许多情况。

## 执行过程

我们先在不考虑优化的情况下谈论执行过程（即严格按照`SI`的含义，对点查询取一个单独的快照来执行），`TiKV`是一个分布式`KV`数据库，即想要在`TiKV`中查出值来，我们首先得有一个`key`，所以首先得在`TiDB`里面把`Key`给构造出来。

这里以`point(id int primary key, c int, d varchar(10), unique c_d (c, d))`表为例，有两种构造`key`的方式。

```SQL
select * from point where id = 1 and c = 0; -- key: 74800000000000002d5f728000000000000001
select * from point where id < 0 and c = 1 and d = 'b'; -- key: 74800000000000002d5f698000000000000001038000000000000001016200000000000000f8
```

这两条`SQL`语句的区别在于是否能够通过主键直接确定是哪一条记录，所以构造出来的`key`也不完全一样

将`key`分解为几部分来看

```
74800000000000002d 5f728000000000000001
74800000000000002d 5f698000000000000001 038000000000000001016200000000000000f8
```

## Internal Read

考虑这一种情况。

```SQL
create table t(id int primary key, val int);
begin;
insert into t values(1, 1);
select * from t where id = 1;
```

在这个
