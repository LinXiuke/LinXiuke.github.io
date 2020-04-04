---
title: MySQL事务日志

date: 2020-03-24

categories: 
- MySQL
---

### bin log  （二进制日志）

比如我们如果在MySQL中修改了一条记录，而用户检索出来的数据是通过MySQL的搜索引擎的，为了保证能检索到最新的数据，需要把搜索引擎的数据也一起改掉。

bin log记录了数据库表结构和表数据变更的记录，比如create/insert/update/delete等操作，但是select不会记录，因为select不存在修改表的操作。bin log记录着每条变更的SQL已经事务id等记录。

<!--more-->

***备份和恢复***

- 比如MySQL使用主从分离的时候，主库和从库之间就是通过bin log来同步数据的。
- 数据库的数据被删了，可以用个binlog恢复


### redo log （重做日志）

MySQL的数据实际是存在“页”中的，当修改一条记录时，MySQL实际先是寻找到这条记录所在的页，然后加载到内存中对数据修改。但是如果在内存中把数据修改了，还没有写到磁盘中，数据库挂了怎么办？这个时候就可以通过redo log恢复操作。MySQL虽然是持久化的，但并不是每一次操作都立刻写入磁盘，因为那样效率太低。而redo log则记录着这次操作对某个页的某数据做了什么修改。

***总结***

redo log存在的意义在于： 当进行某次修改时，内存中操作结束了，数据却还没有写入磁盘的时候数据库挂了，这时候可以用redo log恢复， redo log是顺序IO，写入速度很快，并且记录的是物理修改（某页做了什么修改），文件体积小恢复快。


***bin log和redo log的区别***

1. bin log记录的是SQL语句，redo log记录的是真实的修改内容。 bin log记录的是逻辑变化，redo log记录的是物理变化
2. bin log是用来恢复数据的，它记录着每条SQL语句，当数据库数据都没删除时可以通过bin log恢复。redo log不会记录历史情况，内存中的数据写入磁盘后redo log记录就无效了，redo log是用来做持久化的。


redo log是MySQL的InnoDB引擎所产生的， bin log无论是什么MySQL引擎都会有的。

redo log事务开始的时候，就开始记录每次的变更信息，而binlog是在事务提交的时候才记录。但是MySQL要保证redo log和bin log都记录成功才会提交事务，否则就会回滚（回滚时会将已写入的bin log删除）

### undo log （回滚日志）

undo log的作用是回滚和MVCC（多版本并发控制）

数据修改的时候用bin log和redo log来记录修改，如果事务执行失败需要回滚，那就需要用undo log了。 

undo log也是逻辑日志， 当执行一条insert语句，undo log会写入一条delete语句，当执行一条update语句时，也会写入一条相反的update语句，用来回滚。

undo log存储的修改之前的数据，相当于前一个版本，所以MVCC读取的时候是要读取前一个版本的数据不会阻塞。