---
title: InnoDB的锁机制
date: 2016-11-10 11:48:42
tags:
  - InnoDB
  - 锁
categories: 编程
---

## 写在前面

使用数据库时，想要较高的吞吐、较低的延迟，但又想在高并发下可以一致地读写数据，因此需要高效的锁机制。

InnoDB中的锁可以分为：

- latch：程序上的锁机制，用来锁定内部对象，没有死锁检测；
- lock：用来锁定数据库中的对象，比如表、页、行，有死锁检测机制；

可以使用`show engine innodb mutex`来查看latch的情况（一般不怎么关心），下面我们来重点看lock。

## 锁机制

### 基础概念

锁的目的是将一个资源占住，不让其他人操作。更准确地可以描述为：

> 禁止其他人对某个资源执行某个操作。

数据库操作无非是读、写，那么相应的锁类型为：

- 共享锁（S）：允许事务读数据；
- 排他锁（X）：允许事务修改数据；

锁并非都互斥（有你没他），比如两个事务可以对同一条记录加S锁来读数据。共存即兼容：

 兼容性|X|S
---|---|---
X|不兼容|不兼容
S|不兼容|兼容

此外还支持一种额外的锁方式（意向锁）：

- 意向共享锁（IS）
- 意向排他锁（IX）

其含义是：

> 在行上加普通锁之前，需要在更粗的粒度上加对应类型的意向锁，也就是意向锁反映了行锁的情况，能提升加锁效率。

比如，事务A想对表加S锁，需要判断表中是否有行有X锁。以前需要逐行判断，而现在直接判断表上有没有IX锁即可。数据库中层次结构如下：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/e04d28cc04236a8df844ba2cb55e1672/image.png)

包含意向锁的兼容性为：

 兼容性|IS|IX|S|X
---|---|---|---|---
IS|兼容|兼容|兼容|不兼容
IX|兼容|兼容|不兼容|不兼容
S|兼容|不兼容|兼容|不兼容
X|不兼容|不兼容|不兼容|不兼容

### 加锁算法

不同算法的区别在于范围：

1. Record Lock：锁定一行记录；
2. Gap Lock：锁定一个范围，不包含记录；
3. Next-Key Lock：锁定一个范围，包含记录；

### 死锁

在两个及以上的事务在执行过程中，可能因为争夺资源而互相等待：

- T1拥有资源A，尝试获取资源B；
- T2拥有资源B，尝试获取资源A；

如果不暴力终结某个事务，T1、T2会永远等下去，这就是死锁。数据库通过wait-for graph（等待图）来检测是否存在死锁，当发现有回路时，引擎会选择回滚undo量最小的事务。

程序上为防止死锁一般会统一更新顺序。

## 加锁分析

下面对常见的几条SQL做加锁分析，建表如下：

```sql
CREATE TABLE `my_test` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `a` bigint(20) NOT NULL,
  `b` bigint(20) NOT NULL,
  `c` bigint(20) NOT NULL,
  `d` bigint(20) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `unique_a_b`(`a`,`b`),
  KEY `idx_c`(`c`)
);
```

初始化数据如下：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/ef7441a47ac1c49f5add86f2a905552c/image.png)

在执行完成SQL后，可以从__information_schema__中的：

- INNODB_TRX
- INNODB_LOCKS
- INNODB_LOCK_WAITS

来查看锁和事务的状态，此外也可以用`show engine innodb status`来看整体情况，需要增加锁监控：

```sql
create table innodb_lock_monitor(x int) engine=innodb;
```

才可以看到比较详细的锁信息。

### RC+主键

```sql
select * from my_test where id = 1 for update;
```

加锁信息：

```
2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 2, OS thread handle 0x1648, query id 148 localhost 127.0.0.1 root cleaning up
TABLE LOCK table `mysql`.`my_test` trx id 2863 lock mode IX
RECORD LOCKS space id 38 page no 3 n bits 72 index `PRIMARY` of table `mysql`.`my_test` trx id 2863 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
```

加锁逻辑为：

1. 表上加IX锁；
2. 行上加X锁；

### RC+唯一约束

```sql
select * from my_test where a = 1 and b = 1 for update;

```

加锁信息如下：

```
3 lock struct(s), heap size 360, 2 row lock(s)
MySQL thread id 2, OS thread handle 0x1648, query id 152 localhost 127.0.0.1 root cleaning up
TABLE LOCK table `mysql`.`my_test` trx id 2864 lock mode IX
RECORD LOCKS space id 38 page no 4 n bits 72 index `unique_a_b` of table `mysql`.`my_test` trx id 2864 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 8; hex 8000000000000001; asc         ;;
 2: len 8; hex 8000000000000001; asc         ;;

RECORD LOCKS space id 38 page no 3 n bits 72 index `PRIMARY` of table `mysql`.`my_test` trx id 2864 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500110; asc     P  ;;
 3: len 8; hex 8000000000000001; asc         ;;
 4: len 8; hex 8000000000000001; asc         ;;
 5: len 8; hex 8000000000000001; asc         ;;
 6: len 8; hex 8000000000000001; asc         ;;
```

加锁逻辑为：

1. 表上加IX锁；
2. 唯一约束上加X锁；
3. 行上加X锁；

### RC+普通索引

```sql
select * from my_test where c = 3 for update;
```

加锁信息如下：

```
3 lock struct(s), heap size 360, 5 row lock(s)
MySQL thread id 2, OS thread handle 0x1648, query id 166 localhost 127.0.0.1 root cleaning up
TABLE LOCK table `mysql`.`my_test` trx id 2872 lock mode IX
RECORD LOCKS space id 38 page no 5 n bits 72 index `idx_c` of table `mysql`.`my_test` trx id 2872 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 8; hex 8000000000000003; asc         ;;
 1: len 8; hex 8000000000000003; asc         ;;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 8; hex 8000000000000003; asc         ;;
 1: len 8; hex 8000000000000004; asc         ;;

RECORD LOCKS space id 38 page no 3 n bits 72 index `PRIMARY` of table `mysql`.`my_test` trx id 2872 lock_mode X locks rec but not gap
Record lock, heap no 4 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000003; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500130; asc     P 0;;
 3: len 8; hex 8000000000000003; asc         ;;
 4: len 8; hex 8000000000000003; asc         ;;
 5: len 8; hex 8000000000000003; asc         ;;
 6: len 8; hex 8000000000000003; asc         ;;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000004; asc         ;;
 1: len 6; hex 000000000b37; asc      7;;
 2: len 7; hex ac0000015e0110; asc     ^  ;;
 3: len 8; hex 8000000000000004; asc         ;;
 4: len 8; hex 8000000000000004; asc         ;;
 5: len 8; hex 8000000000000003; asc         ;;
 6: len 8; hex 8000000000000004; asc         ;;
```

加锁逻辑为：

1. 表上加IX锁；
2. 索引上加X锁（2）；
3. 行上加X锁（2）；

### RC+无索引

```sql
select * from my_test where d = 1 for update;
```

加锁信息：

```
2 lock struct(s), heap size 360, 5 row lock(s)
MySQL thread id 2, OS thread handle 0x1648, query id 172 localhost 127.0.0.1 root cleaning up
TABLE LOCK table `mysql`.`my_test` trx id 2874 lock mode IX
RECORD LOCKS space id 38 page no 3 n bits 72 index `PRIMARY` of table `mysql`.`my_test` trx id 2874 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500110; asc     P  ;;
 3: len 8; hex 8000000000000001; asc         ;;
 4: len 8; hex 8000000000000001; asc         ;;
 5: len 8; hex 8000000000000001; asc         ;;
 6: len 8; hex 8000000000000001; asc         ;;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500120; asc     P  ;;
 3: len 8; hex 8000000000000002; asc         ;;
 4: len 8; hex 8000000000000002; asc         ;;
 5: len 8; hex 8000000000000002; asc         ;;
 6: len 8; hex 8000000000000002; asc         ;;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000003; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500130; asc     P 0;;
 3: len 8; hex 8000000000000003; asc         ;;
 4: len 8; hex 8000000000000003; asc         ;;
 5: len 8; hex 8000000000000003; asc         ;;
 6: len 8; hex 8000000000000003; asc         ;;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000004; asc         ;;
 1: len 6; hex 000000000b37; asc      7;;
 2: len 7; hex ac0000015e0110; asc     ^  ;;
 3: len 8; hex 8000000000000004; asc         ;;
 4: len 8; hex 8000000000000004; asc         ;;
 5: len 8; hex 8000000000000003; asc         ;;
 6: len 8; hex 8000000000000004; asc         ;;
```

加锁逻辑：

1. 表上加X锁；
2. 行上加X锁（4）；

### RR

在事务1中执行：

```sql
delete from my_test where c = 3;
```

在事务2中执行：

```sql
insert into my_test values(6, 6, 6, 3, 6);
```

查看事务2的锁信息如下：

```
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1
MySQL thread id 3, OS thread handle 0x2378, query id 198 localhost 127.0.0.1 root update
insert into my_test values(6, 6, 6, 3, 6)
------- TRX HAS BEEN WAITING 20 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 38 page no 5 n bits 72 index `idx_c` of table `mysql`.`my_test` trx id 2897 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

查看锁等待信息`select * from information_schema.innodb_locks`如下：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/ae0f37aa7431015b6183083100439712/image.png)

而对于__lock_data__：

> If a gap lock is taken for key values or ranges above the largest value in the index, LOCK_DATA reports “supremum pseudo-record".

根据上面的__Next-Key__算法分析，锁定的不只是单个记录，而是一个范围：[3, 4)。

## 死锁分析

首先模拟常见的死锁，事务1：

```sql
begin;
select * from my_test where id = 1 for update;
```

事务2：

```sql
begin;
select * from my_test where id = 2 for update;
```

事务1：

```sql
select * from my_test where id = 2 for update; 
```

此时处于锁等待，接着事务2：

```sql
select * from my_test where id = 1 for update; 
```

此时事务2会报死锁，然后将事务回滚，查看死锁信息如下：

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2016-11-22 21:45:37 2378
*** (1) TRANSACTION:
TRANSACTION 2899, ACTIVE 48 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 360, 2 row lock(s)
MySQL thread id 2, OS thread handle 0x1648, query id 217 localhost 127.0.0.1 root statistics
select * from my_test where id = 2 for update
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 38 page no 3 n bits 80 index `PRIMARY` of table `mysql`.`my_test` trx id 2899 lock_mode X locks rec but not gap waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500120; asc     P  ;;
 3: len 8; hex 8000000000000002; asc         ;;
 4: len 8; hex 8000000000000002; asc         ;;
 5: len 8; hex 8000000000000002; asc         ;;
 6: len 8; hex 8000000000000002; asc         ;;
*** (2) TRANSACTION:
TRANSACTION 2900, ACTIVE 28 sec starting index read, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
3 lock struct(s), heap size 360, 2 row lock(s)
MySQL thread id 3, OS thread handle 0x2378, query id 218 localhost 127.0.0.1 root statistics
select * from my_test where id = 1 for update
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 38 page no 3 n bits 80 index `PRIMARY` of table `mysql`.`my_test` trx id 2900 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500120; asc     P  ;;
 3: len 8; hex 8000000000000002; asc         ;;
 4: len 8; hex 8000000000000002; asc         ;;
 5: len 8; hex 8000000000000002; asc         ;;
 6: len 8; hex 8000000000000002; asc         ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 38 page no 3 n bits 80 index `PRIMARY` of table `mysql`.`my_test` trx id 2900 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 000000000b23; asc      #;;
 2: len 7; hex 9c000001500110; asc     P  ;;
 3: len 8; hex 8000000000000001; asc         ;;
 4: len 8; hex 8000000000000001; asc         ;;
 5: len 8; hex 8000000000000001; asc         ;;
 6: len 8; hex 8000000000000001; asc         ;;

*** WE ROLL BACK TRANSACTION (2)
```

从死锁信息上能看到事务分别拥有哪些锁、在申请哪些锁的时候互相等待了。

### INSERT + Unique Key

还是沿用上面的表，开启事务1：

```sql
begin;
insert into my_test values (7, 7, 7, 0, 0);
```

开启事务2：

```sql
begin;
insert into my_test values (8, 7, 7, 0, 0);
```

此时会锁等待，再开启事务3：

```sql
begin;
insert into my_test values (9, 7, 7, 0, 0);
```

此时的锁情况为：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/3e2600f5b5b51962a70c297312eb01d3/image.png)

原因：

> 事务1在插入是需要加X锁，而事务2、事务3在插入前需要做唯一性检查，所以需要加S锁，因为与X锁不兼容所以事务2、3处于等待状态。

接着事务1进行rollback话，事务2会插入成功，事务3会报死锁：

```
2016-11-22 22:36:36 1ea4
*** (1) TRANSACTION:
TRANSACTION 2904, ACTIVE 300 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 3, OS thread handle 0x2378, query id 238 localhost 127.0.0.1 root update
insert into my_test values (8, 7, 7, 0, 0)
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 38 page no 4 n bits 72 index `unique_a_b` of table `mysql`.`my_test` trx id 2904 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 2905, ACTIVE 290 sec inserting, thread declared inside InnoDB 1
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 6, OS thread handle 0x1ea4, query id 239 localhost 127.0.0.1 root update
insert into my_test values (9, 7, 7, 0, 0)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 38 page no 4 n bits 72 index `unique_a_b` of table `mysql`.`my_test` trx id 2905 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 38 page no 4 n bits 72 index `unique_a_b` of table `mysql`.`my_test` trx id 2905 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** WE ROLL BACK TRANSACTION (2)
```

原因：

> 在事务1回滚后，事务2、3都获得了S锁，接着要执行INSERT要去获取X锁，事务2申请X锁时发现与事务3的S锁不兼容，事务3也是同样的情况，于是出现死锁。

特殊的唯一约束：__主键__也有相同的问题。

## 总结

对InnoDB了解的还是很少，最近把锁相关的突击了一下，抽时间把底层的实现也了解一下，其他部分再慢慢补上。