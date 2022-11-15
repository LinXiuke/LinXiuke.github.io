---

title: InnoDB-MVCC

date: 2020-11-14

categories:
- MySQL

---

MVCC机制的全称为Multi-Version Concurrency Control，即多版本并发控制技术， 主要是为了提升数据库并发性能而设计的，其中采用更好的方式处理了读-写并发冲突，做到即使有读写冲突时，也可以不加锁解决，从而确保了任何时刻的读操作都是非阻塞的。

\<!--more-->

## MVCC机制的组成

MVCC机制主要通过隐藏字段、undo log和ReadView来实现的

### 隐藏字段

在InnoDB引擎中主要有DB\_ROW\_ID、DB\_TRX\_ID、DB\_ROLL\_PTR这三个隐藏字段

#### DB\_ROW\_ID

对于InnoDB引擎的表，表数据是按照聚簇索引的格式存储的， 一般会采用主键作为聚簇索引列并构建索引树，如果表没有定义主键则会选择一个具备非空唯一属性的字段来作为聚簇索引字段。

如果未定义主键也不具备唯一非空属性字段，InnoDB就会隐式定义一个递增的DB\_ROW\_ID作为主键

#### DB\_TRX\_ID

DB\_TRX\_ID即事务id，MySQL对于每一个事务都会分配一个事务id，并且事务id是循序递增的，即后开启的事务id一定比先开启的事务id要大。

如果是一条select语句，事务id为0，但是如果是手动开启的事务，即便是有一条select语句也会分配递增的事务id。

#### DB\_ROLL\_PTR

ROLL\_PTR全称为rollback\_pointer，即回滚指针，当对一条数据做了改动后，会把旧版本的数据放到undo log中，而DB\_ROLL\_PTR则指向undo log中的旧版本数据，当事务需要回滚时则通过该字段找回旧数据。

### undo log

MySQL的事务回滚时基于undo log实现的，undo log中存储了数据的旧版本，而且是链式的，比如一个事务对一条数据修改了两次，那么undo log中就会有两条旧版本数据，最新的旧版本数据会在链表头部。

### ReadView

MVCC的意思是多版本并发控制， InnoDB通过undo log实现数据的多版本，而并发控制则是通过ReadView来实现。

当一个事务在读取一条数据时，MVCC根据当前MySQL的运行状态生成的快照，就是ReadView。

Review的核心内容有四个：

*   creator\_trx\_id： 创建这个ReadView的事务id
*   trx\_ids： 生成这个ReadView还活跃着的其它事务id列表
*   up\_limit\_id： trx\_ids中的最小值
*   low\_limit\_id: 生成该ReadView时，系统要给下一个事务分配的id

## MVCC的实现原理

当前有两个事务T1和T2，T1正在修改ID=1的数据，而T2要查询这条数据，怎么判断T2能否读取到最新数据：

1.  当开始查询时，会生成一个ReadView
2.  判断数据中DB\_*TRX*\_ID == ReadView\.creator\_trx\_id，相同则代表创建ReadView的事务和当前修改事务是同一个，**可以读取最新数据**
3.  判断DB\_*TRX*\_ID >= ReadView\.low\_limit\_id，是，则说明修改数据的事务在创建ReadView后生成，**不能读取到最新数据**
4.  ReadView\.up\*\_low\_id <= *DB\_*TRX\*\_ID < ReadView\.low\_limit\_id，判断DB\_*TRX\_ID是否在trx*\_ids中

    *   在：修改数据的事务还在执行，**不能读取最新数据**
    *   不在：修改事务的数据已结束，**可以读取最新数据**
5.  DB\_*TRX*\_ID < ReadView\.up\_limit\_id（想了想到了判断这一步一定是小于的），是，则说明修改数据的事务在创建ReadView前已经提交，**可以读取最新数据**

上述的345三点，可以把up\_limit\_id和low\_limit\_id看成*trx*\_ids的左右区间，DB\_*TRX*\_ID在区间左（不包含）可以读取最新数据，区间右（包含）不可以读取最新数据， 在区间内的，则判断DB\_*TRX*\_ID是否在*trx*\_ids内。

之所以还要判断DB\_*TRX*\_ID是否在*trxids内，是因为如果有T3、T4、T5事务存在，当T3回滚，则trx*\_ids里不包含T3， 或当T5已提交，*trx*\_ids里不包含T5。

## 不同隔离级别下的MVCC

读未提交：数据直接修改读取，无MVCC

串行化：串行化执行，无并行化所以也没有MVCC

使用到MVCC的是读已提交和可重复读这两个隔离界别

*   读已提交：每次select语句生成一个ReadView
*   可重复读：首次select语句生成ReadView

## MVCC与幻读问题

现在user表存在如下数据：

| id | name |
| :- | :--- |
| 1  | A1   |

当开启事务T1查询

```sql
// 步骤1
// 开启事务但是先不提交
begin;
select * from user where id > 1;

// 得到结果
Empty set
```

新开一个连接插入一条数据

```sql
// 步骤2
// 没有手动开启事务单条SQL即为一个事务
insert into user(id, name) value(2, 'A2');
```

这个时候继续在T1事务下执行

```sql
// 步骤3

select * from user where id > 1;

// 得到结果集还是未空
Empty set

// 这好像是很正常的，MVCC不就是这样吗
// 继续执行
update user set name = 'A21' where id = 2;
select * from user where id > 1;
// 这次查询得到的结果
+----+------+
| id | name |
+----+------+
|  2 | A21  |
+----+------+
1 row in set (0.00 sec)
```

先更新id=2的数据然后再查询发现又能查询到了，第三次查到的结果集和前两次不一样，这不就是幻读的定义吗。&#x20;

之所以能查到，是因为第三次去查询的时候id=2的行数据DB\_*TRX*\_ID就是T1本身，所以又能查询到了。

还有一种情况，我们把数据恢复到最开始的状态，通用执行步骤1、2，步骤3改成如下

```sql
// 步骤3
 select * from user where id > 1 lock in share mode;
+----+------+
| id | name |
+----+------+
|  2 | A2   |
+----+------+
1 row in set (0.00 sec)
```

这个时候发现也能查询到数据， 这是因为MVCC是基于读取时的无锁优化，当加了锁以后则不会通过MVCC方式读取，从快照读改为了当前读。同理 select ... for update、 insert、update和delete都是当前读

## 参考

[掘金文章：MySQL之MVCC机制](https://juejin.cn/post/7155359629050904584#heading-1)

[凤凰架构](http://icyfenix.cn/architect-perspective/general-architecture/transaction/local.html)

《MySQL技术内部：InnoDB存储引擎》
