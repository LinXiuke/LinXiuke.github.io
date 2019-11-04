---
title: 事务

date: 2019-10-31

categories: 
- MySQL

tags:
- Transaction
---

## 什么是事务

简单的说，事务就是一组原子性的SQL，这一组 SQL 要么全部执行成功，要么全部执行失败。

一般来说，事务是必须满足4个条件（ACID）：：原子性（Atomicity，或称不可分割性）、一致性（Consistency）、隔离性（Isolation，又称独立性）、持久性（Durability）。


<!--more-->


- 原子性：一个事务中的操作，要么全部完成，要么全部不完成，事务在执行过程中出现错误就会回滚到事务执行之前的状态
- 一致性：在事务开始之前和事务结束之后，数据库的完整性没有被破坏，即前后保持一致
- 隔离性：通常来说，一个事务提交之前对其他事务是不可见的，但是这里所说的不可见需要考虑隔离级别，比如未提交读在提交前对于其他事务来说也是可见的。
- 持久性：事务一旦被提交，那么对数据库的修改会被永久的保存，即使数据库崩溃修改后的数据也不会丢失。


## 事务的隔离级别

SQL标准定义的四个隔离级别为

1. READ UNCOMMITTED
2. READ COMMITTED
3. REPEATABLE READ
4. SERIALIZABLE


### READ UNCOMMITTED 

READ UNCOMMITTED（读取未提交）

在该隔离级别，所有事务都可以看到其他未提交(commit)事务的执行结果。

读取未提交的数据，也被称之为脏读（Dirty Read）。


### READ COMMITTED 

READ COMMITTED（读取已提交）

大多数的数据库事务默认隔离级别（但不是MySQL默认的）。

满足隔离的简单定义：一个事务只能看到已经提交的事务做出的改变。
支持不可重复读，因为期间可能会有新的commit所有同一查询语句可能获取到两个不同结果


### REPEATABLE READ

REPEATABLE READ（可重复读）

MySQL的默认事务隔离级别，支持同一事务的多个实例在并发读取时获取相同的数据行。但是在不同事务下可能出现幻读。InnoDB存储引擎使用Next-Key Lock锁的算法避免了幻读


### SERIALIZABLE

SERIALIZABLE（序列化）

这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。