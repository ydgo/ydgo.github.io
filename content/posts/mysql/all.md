---
title: "All"
date: 2023-09-25T10:09:06+08:00
draft: true
categories: ["MySQL"]
tags: [mysql]
---

<!--more-->

## InnoDB

在 MySQL 5.7中，InnoDB 是新表的默认存储引擎。

![InnoDB 架构图5.7](/img/innodb-architecture.png)


### 基本特性

事务（提交、回滚、崩溃恢复：InnoDB 崩溃恢复会自动完成在崩溃之前提交的更改，并撤销正在进行但尚未提交的更改）

行级锁

聚集索引的主键索引，优化查询IO

外键约束

全文索引

MVCC

存储限制64TB






自适应散列索引

online ddl

InnoDB 行格式


INFORMATION_SCHEMA

Performance Schema

### 最佳实践

- 主键列
- 关闭自动提交（除非显式地开始一个事务，否则每个查询都被当做一个单独的事务自动执行），每秒提交数百次会限制性能（默认开启）
- 不要使用 LOCK TABLES 语句。InnoDB 可以同时处理对同一个表的多个读写会话，而不会牺牲可靠性或高性能。若要获得对一组行的独占写访问权，请使用 SELECT... FOR UPDATE 语法仅锁定要更新的行。

### ACID

- A: atomicity
- C: consistency
- I: isolation
- D: durability

#### 原子性

提交
回滚

#### 一致性

doublewrite buffer(https://dev.mysql.com/doc/refman/5.7/en/innodb-doublewrite-buffer.html)

参考(https://www.modb.pro/db/114783)

主要作用：

InnoDB 页大小默认：16kb

Linux文件系统页大小：4kb

所以InnoDB的页会对应4个磁盘页

InnoDB 页数据落盘到磁盘时，可能会掉电，导致只同步了3个页，就会造成页损坏。解决方案就是：Doublewrite buffer

Doublewrite buffer 是一个存储区域，InnoDB 在将页写入到 InnoDB 数据文件中的适当位置之前，将从缓冲池中刷新的页写入。
如果在页面写入过程中出现操作系统、存储子系统或意外的 mysqld 进程退出，InnoDB 可以在崩溃恢复期间从双写缓冲区找到该页面的良好副本。



crash recovery. 崩溃恢复(https://dev.mysql.com/doc/refman/5.7/en/innodb-recovery.html#innodb-crash-recovery)

InnoDB 自动回滚崩溃时存在的未提交事务



#### 隔离性

隔离级别

未提交读
提交读
可重复读
串行化

### MVCC

InnoDB 是一个多版本存储引擎。它保存关于已更改行的旧版本的信息，以支持**并发**和**回滚**等事务特性

每行添加3个字段：

- DB_TRX_ID： 插入或更新该行的最后一个事务的事务标识符。此外，删除在内部被视为更新，其中设置行中的特殊位以将其标记为已删除。
- DB_ROLL_PTR： 回滚指针指向写入回滚段的撤消日志记录。如果该行已更新，撤消日志记录将包含在更新之前重新生成该行内容所需的信息。
- DB_ROW_ID： 该行 ID 随着新行的插入而单调增加。如果InnoDB自动生成聚集索引，则该索引包含行ID值。否则，DB_ROW_ID 列不会出现在任何索引中。


在 InnoDB 多版本控制方案中，当您使用 SQL 语句从数据库中删除某行时，该行不会立即从数据库中物理删除。
InnoDB 在丢弃为删除而编写的更新撤消日志记录时，仅物理删除相应的行及其索引记录。
这种删除操作称为 purge，速度非常快，通常与执行删除操作的 SQL 语句的时间顺序相同。




## 索引

## 锁



## DML

## DDL

## 主从复制（服务器层面）


## 调优

### 缓冲池

缓存表数据与索引数据，把磁盘上的数据加载到缓冲池，避免每次访问都进行磁盘IO，起到加速访问的作用。

就是在内存维护一个包含多行的页列表，列表头部是最近访问的，列表尾部是最近最少访问的。

我们可以通过以下配置来优化缓冲池：

- 设置尽可能大的缓冲池大小
- 配置多个缓冲池实例
- 使缓冲池扫描具有抗性，避免因为流量高峰将不常访问的数据带入缓冲池，从而将常访问的数据驱逐出缓冲池。
- 配置 InnoDB 缓冲池预取(预读)，通过预读将磁盘的数据带入
- 配置缓冲池刷新
- 保存和恢复缓冲池状态，避免服务器重新启动后漫长的预热期

我们可以通过 SHOW ENGINE INNODB STATUS 来使用 InnoDB 标准监视器监视缓冲池，输出如下

~~~text
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 94124
Buffer pool size   8192
Free buffers       7804
Database pages     386
Old database pages 0
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 352, created 34, written 36
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 386, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]

~~~

缓冲池：在专用的数据库服务器上，高达80% 的物理内存通常分配给缓冲池（https://www.modb.pro/db/111341）

了解如何利用缓冲池将频繁访问的数据保留在内存中是MySQL调优的一个重要方面。

https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool.html

思考：

1. 从操作系统和mysql的缓存架构去看应用中的缓存架构如何实现？（避免每次查询数据都去DB查询）

分级，淘汰算法，如何更新

访问快，但容量小

把最热的数据放到最近的地方（是存最热还是存业务用的全部数据）

预读

操作系统的磁盘读写，并不是按需读取，而是按页读取，一次至少读一页数据（一般是4K），如果未来要读取的数据就在页中，就能够省去后续的磁盘IO，提高效率。

数据访问，通常都遵循“集中读写”的原则，使用一些数据，大概率会使用附近的数据，这就是所谓的“局部性原理”，它表明提前加载是有效的，确实能够减少磁盘IO。

用户请求任务时，将用户相关的任务数据预读到缓存中

mysql中的缓冲池使用的LRU与传统的LRU算法不太一样，原因如下：

一、预读失效：由于预读（Read-Ahead），提前把页放入了缓冲池，但最终MySQL并没有从页中读取数据，称为预读失效。
二、缓冲池污染：当某一个SQL语句，要批量扫描大量数据时，可能导致把缓冲池的所有页都替换出去，导致大量热数据被换出，MySQL性能急剧下降，这种情况叫缓冲池污染。

### change buffer（写缓冲）

经典操作：

减少磁盘/网络IO操作

缓冲操作

1. 将随机写转换为顺序写
2. 异步、批量同步数据（批量写是常见的优化手段）
3. 但是要保证数据一致性和崩溃恢复

脏页：内存中的页数据和磁盘的页数据不一致称为脏页，待脏页刷新到磁盘后，则数据一致

https://www.modb.pro/db/112469

问题本质：

写入数据如何减少IO？写入数据如何多写内存少写磁盘？


插入、更新和删除通过称为更改缓冲的自动机制进行优化。InnoDB 不仅允许对同一个表进行并发的读写访问，而且还可以缓存已更改的数据以简化磁盘 I/O。

适用场景

- 写多读少
- 大部分表是非唯一索引（因为对有唯一索引的表的写操作需要进行唯一性校验，所以读IO是必不可少的）


一、假如要修改的页4正好在缓冲池内

1、直接修改缓冲池中的页，一次内存操作；
2、写入redo log，一次磁盘顺序写操作；
这样的效率是最高的（像写日志这种顺序写，每秒几万次没问题）。

二、假如要修改的这个页40正好不在缓冲池内。

1、先把需要为40的索引页，从磁盘加载到缓冲池，一次磁盘随机读操作；

2、修改缓冲池中的页，一次内存操作；

3、写入redo log，一次磁盘顺序写操作；

没有命中缓冲池的时候，至少产生一次磁盘IO，对于写多读少的业务场景，是否还有优化的空间呢？

这即是InnoDB考虑的问题，写缓冲(change buffer)应用而生（从名字容易看出，写缓冲是降低磁盘IO，提升数据库写性能的一种机制）。

在非唯一普通索引页（non-unique secondary index page）不在缓冲池中，对页进行了写操作，并不会立刻将磁盘页加载到缓冲池，而仅仅记录缓冲变更（Buffer Changes），
等未来数据被读取时，再将数据合并（Merge）恢复到缓冲池中的技术。写缓冲的目的是降低写操作的磁盘IO，提升数据库性能。

加入写缓冲优化后，流程优化为：

1、在写缓冲中记录这个操作，一次内存操作；

2、写入redo log，一次磁盘顺序写操作；

其性能与这个索引页在缓冲池中，相近（可以看到，40这一页，并没有加载到缓冲池中）。

是否会出现一致性问题呢？
也不会。
1、数据库异常奔溃，能够从redo log中恢复数据；
2、写缓冲不只是一个内存结构，它也会被定期刷盘到写缓冲系统表空间；
3、数据读取时，有另外的流程，将数据合并到缓冲池；






## 提问

### 大事务

什么是大事务： 运行时间比较长，长时间未提交的事务就是大事务。

大事务产生的原因： 操作的数据比较多、大量的锁竞争，其他非DB的耗时操作

会造成什么后果： 占用连接；锁定太多数据，造成大量的阻塞和锁超时；执行时间长，容易造成主从延迟；回滚时间比较长；undo log暴涨

如何查询大事务： 以查询执行时间超过10m的事务为例：
~~~mysql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>10
~~~

如何避免： 避免一次处理太多数据、尽量避免不必要的查询、避免耗时操作，以下：

在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放

监控 information_schema.Innodb_trx表，设置长事务阈值，超过就报警/或者kill

