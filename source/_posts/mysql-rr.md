---
title: MySQL幻读验证
date: 2019-11-14 11:49:09
tags: [幻读]
categories: MySQL
---

# 前言

我们都知道MySQL的默认隔离级别是REPEATABLE-READ（可重复读）：

```mysql
select @@global.tx_isolation;
```

之前有看到文章上说，MySQL的默认隔离级别RR已经可以阻止幻读的发生，但是没有验证过，现在就来验证一下MySQL隔离级别RR下是否可以阻止幻读的发生。

# 正文

首先我们了解一下幻读的定义：事务A读取与搜索条件相匹配的若干行。事务B以插入或删除行等方式来修改事务A的结果集，然后再提交。 事务A再次以相同的条件读取，会发现结果不一致（增加或减少）。和不可重复读相比幻读更侧重于插入或删除操作。

## RC隔离级别出现幻读

1、先将两个会话的隔离级别调整为RC

```mysql
mysql> set tx_isolation ='read-committed';
Query OK, 0 rows affected (0.02 sec)

mysql> select @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| READ-COMMITTED |
+----------------+
1 row in set (0.04 sec)
```

2、会话1执行：

```mysql
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> select * from mysql_learn;
+----+--------+
| id | number |
+----+--------+
|  1 |     10 |
|  3 |     11 |
|  4 |     15 |
|  5 |     11 |
| 30 |     11 |
+----+--------+
5 rows in set (0.05 sec)
```

3、会话2执行：

```mysql
mysql>  insert into mysql_learn value(32,11);
Query OK, 1 row affected (0.03 sec)
```

可以看到插入成功了。

4、会话1再执行查询：

```mysql
mysql> select * from mysql_learn;
+----+--------+
| id | number |
+----+--------+
|  1 |     10 |
|  3 |     11 |
|  4 |     15 |
|  5 |     11 |
| 30 |     11 |
| 32 |     11 |
+----+--------+
6 rows in set (0.04 sec)

mysql> commit;
Query OK, 0 rows affected (0.02 sec)
```

可以看出产生了幻读现象。**在RC的隔离级别下，即使是快照读也是读到已提交事务写入的最新的数据。**

如果使用`select for update`（当前读）呢：

1、会话1执行：

```mysql
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> select * from mysql_learn where number = 11 for update;
+----+--------+
| id | number |
+----+--------+
|  3 |     11 |
|  5 |     11 |
| 30 |     11 |
| 32 |     11 |
+----+--------+
4 rows in set (0.04 sec)
```

2、会话2执行：

```mysql
mysql> update mysql_learn set number = 15 where id = 30;
1205 - Lock wait timeout exceeded; try restarting transaction
```

更新语句被阻塞无法执行，说明已经添加记录锁。但是插入语句还是可以执行：

```mysql
mysql>  insert into mysql_learn value(35,11);
Query OK, 1 row affected (0.03 sec)
```

3、会话1再执行查询：

```mysql
mysql> select * from mysql_learn where number = 11 for update;
+----+--------+
| id | number |
+----+--------+
|  3 |     11 |
|  5 |     11 |
| 30 |     11 |
| 32 |     11 |
| 35 |     11 |
+----+--------+
5 rows in set (0.04 sec)

mysql> commit;
Query OK, 0 rows affected (0.02 sec)
```

可以看出还是发生了幻读现象，从中可以看出使用RC隔离级别下加锁可以避免不可重复读现象但是还是无法避免幻读。

## RR隔离级别避免幻读

将会话的隔离级别调整回RR。首先使用快照读：

1、会话1执行：

```mysql
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> select * from mysql_learn;
+----+--------+
| id | number |
+----+--------+
|  1 |     10 |
|  3 |     11 |
|  4 |     15 |
|  5 |     11 |
| 30 |     11 |
| 32 |     11 |
| 35 |     11 |
+----+--------+
7 rows in set (0.04 sec)
```

2、会话2执行：

```mysql
mysql>  insert into mysql_learn value(36,11);
Query OK, 1 row affected (0.03 sec)
```

3、会话1再执行查询：

```mysql
mysql> select * from mysql_learn;
+----+--------+
| id | number |
+----+--------+
|  1 |     10 |
|  3 |     11 |
|  4 |     15 |
|  5 |     11 |
| 30 |     11 |
| 32 |     11 |
| 35 |     11 |
+----+--------+
7 rows in set (0.05 sec)
mysql> commit;
Query OK, 0 rows affected (0.02 sec)
```

发现没有产生幻读现象，会话2可以执行插入操作，但是会话1再次执行查询时读取到的并不是最新的数据，而是历史的快照数据。

使用`select for update`加上行锁：

1、会话1执行：

```mysql
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> 
mysql> select * from mysql_learn where number = 11 for update;
+----+--------+
| id | number |
+----+--------+
|  3 |     11 |
|  5 |     11 |
| 30 |     11 |
| 32 |     11 |
| 35 |     11 |
| 36 |     11 |
+----+--------+
6 rows in set (0.05 sec)
```

2、会话2执行：

```mysql
mysql>  insert into mysql_learn value(38,11);
1205 - Lock wait timeout exceeded; try restarting transaction

mysql>  insert into mysql_learn value(38,12);
1205 - Lock wait timeout exceeded; try restarting transaction
```

可以看到执行插入操作被阻塞了，在RC的隔离级别下当前读只添加了记录锁，所以虽然匹配命中的记录锁住了，但是插入操作还是可以进行的。而在RR的隔离级别下，使用了临键锁（记录锁和间隙锁的组合），避免了幻读的产生。而由于MySQL的锁是依赖于索引上的，在number字段没加索引的情况下，间隙锁会退化为表锁。而在number添加索引的情况下，本例中查询条件为`where number = 11`，则锁住的区间为(10,15)。

```mysql
+-----+------------+
| id  | number     |
+-----+------------+
|  42 |          9 |
|   1 |         10 |
|   3 |         11 |
|   5 |         11 |
|  30 |         11 |
|  32 |         11 |
|  35 |         11 |
|  36 |         11 |
|   4 |         15 |
|  43 |         15 |
|  44 |         16 |
+-----+------------+
```

# 小结

- MySQL在RR隔离级别下快照读使用历史快照（MVVC）避免幻读，而当前读使用临键锁避免幻读。
- MySQL的锁是基于索引的，如果某个加锁操作未使用索引会退化为表锁，极度影响效率。

# 其他

MySQL版本：5.6；存储引擎：InnoDB

有关间隙锁和临键锁的具体范围可以参照这篇文章： 

https://juejin.im/post/5b8577c26fb9a01a143fe04e

关于MySQL锁可以查看官方文档：

https://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html 