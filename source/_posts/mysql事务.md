---
title: mysql事务
date: 2020-12-10 13:44:50
categories: 数据库
tags: 事务
---

##  事务的四大特性

- **原子性**:  事务最小工作单位，要么全部成功,要没全部失败。
- **一致性**:  事务开始和结束后，数据库完整性不会被破坏。
- **隔离性**:  不同事务之间互不影响，四种隔离级别为RU(读未提交)、RC(读已提交)、RR(可重复读)、SERIALIZABLE （串行化）。
- **持久性**:  事务提交后,对数据的修改是永久性的，即便系统故障也不会丢失。

###  事务的隔离级别

####  读未提交(Read UnCommitted/RU)

一个事务可以读取到另外一个事务未提交的数据。这种隔离级别是最不安全的，因为未提交的事务存在回滚。

####  读已提交(Read Committed/RC)

一个事务因为读取到另一个事务已提交的修改数据，导致在当前事务的不同时间读取同一条数据获取的结果不一致。

####  可重复读(Repeatable Read/RR)（默认）

当前读取此条数据只可读一次,在当前事务中,不论读取多少次，数据仍然是第一次读取的数据,不会因为在第一次读取之后，其它事务再修改提交此数据而产生改变。

####  串行化

- 事务A和事务B，事务A在操作数据库时，事务B只能排队等待
- 这种隔离级别很少使用，吞吐量太低，用户体验差
- 这种级别可以避免“幻像读”，每一次读取的都是数据库中真实存在数据，事务A与事务B串行，而不并发

####  出现问题

**脏读**：当前事务可以查看到别的事务未提交的数据（侧重点在于别的事务未提交）。

**幻读**：幻读的表象与不可重读的表象都让人"懵逼"，很容易搞混，但是如果非要细分的话，幻读的侧重点在于新增和删除。表示在同一事务中，使用相同的查询语句，第二次查询时，莫名的多出了一些之前不存在数据，或者莫名的不见了一些数据。

**不可重读**：不可重读的侧重点在于更新修改数据。表示在同一事务中，查询相同的数据范围时，同一个数据资源莫名的改变了。

###  不同级别拥有问题

|              | 脏读 | 不可重读 | 幻读 |
| :----------: | :--: | :------: | :--: |
| **读未提交** |  √   |    √     |  √   |
|  **读提交**  |  ×   |    √     |  √   |
|  **可重读**  |  ×   |    ×     |  √   |
|  **串行化**  |  ×   |    ×     |  ×   |

## LBCC

**LBCC，基于锁的并发控制，Lock Based Concurrency Control。**

使用锁的机制,在当前事务需要对数据修改时,将当前事务加上锁,同一个时间只允许一条事务修改当前数据,其他事务务必等待锁释放之后才可以操作。

## MVCC

**MVCC，多版本的并发控制，Multi-Version Concurrency Control。**

使用版本来控制并发情况下的数据问题，在B事务开始修改账户且事务未提交时，当A事务需要读取账户余额时，此时会读取到B事务修改操作之前的账户余额的副本数据，但是如果A事务需要修改账户余额数据就必须要等待B事务提交事务。

**MVCC使得数据库读不会对数据加锁，普通的SELECT请求不会加锁，提高了数据库的并发处理能力**。借助MVCC，数据库可以实现READ COMMITTED，REPEATABLE READ等隔离级别，用户可以查看当前数据的前一个或者前几个历史版本，保证了ACID中的I特性（隔离性)。

#####  InnoDB的MVCC实现逻辑

InnoDB的MVCC是通过在每行记录后面保存两个隐藏的列来实现的。一个保存了行的事务ID(DB_TRX_ID),一个保存了行的回滚指针(DB_ROLL_PT)。每开始一个新的事务,都会自动递增产生一个新ID,事务开始时刻的会把事务ID放到当前事务影响的行事务ID中,当查询时需要用当前事务id和每行记录的事务id做比较。

> MVCC只在REPEATABLE READ和READ COMMITIED两个隔离级别下工作。其他两个隔离级别都和 MVCC不兼容 ，因为READ UNCOMMITIED总是读取最新的数据行，而不是符合当前事务版本的数据行。而SERIALIZABLE则会对所有读取的行都加锁。

**MVCC 在mysql 中的实现依赖的是 undo log(下面会介绍) 与 read view 。**

#####  ReadView

**ReadView**中主要包含当前系统中还有哪些活跃的读写事务，把它们的事务id放到一个列表中，我们把这个列表命名为为**m_ids**。

对于查询时的版本链数据是否看见的判断逻辑：

- 如果被访问版本的 trx_id 属性值小于 m_ids 列表中最小的事务id，表明生成该版本的事务在生成 ReadView 前已经提交，所以该版本可以被当前事务访问。
- 如果被访问版本的 trx_id 属性值大于 m_ids 列表中最大的事务id，表明生成该版本的事务在生成 ReadView 后才生成，所以该版本不可以被当前事务访问。
- 如果被访问版本的 trx_id 属性值在 m_ids 列表中最大的事务id和最小事务id之间，那就需要判断一下 trx_id 属性值是不是在 m_ids 列表中，如果在，说明创建 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建 ReadView 时生成该版本的事务已经被提交，该版本可以被访问。

**举个例子：**

######  READ COMMITTED 隔离级别下的ReadView

**每次读取数据前都生成一个ReadView (m_ids列表)**

![image-20201210173409028](https://jameslin23.gitee.io/2020/12/10/mysql事务/image-20201210173409028.png)

​          这里分析下上面的情况下的ReadView

​         时间点 T5 情况下的 SELECT 语句：

​          当前时间点的版本链：

​       ![image-20201210173458790](https://jameslin23.gitee.io/2020/12/10/mysql事务/image-20201210173458790.png)

此时 SELECT 语句执行，当前数据的版本链如上，因为当前的事务777，和事务888 都未提交，所以此时的活跃事务的ReadView的列表情况 **m_ids：[777, 888]** ，因此查询语句会根据当前版本链中小于 **m_ids** 中的最大的版本数据，即查询到的是 Mbappe

时间点 T8 情况下的 SELECT 语句：

当前时间的版本链情况：

![image-20201210173533273](https://jameslin23.gitee.io/2020/12/10/mysql事务/image-20201210173533273.png)

此时 SELECT 语句执行，当前数据的版本链如上，因为当前的事务777已经提交，和事务888 未提交，所以此时的活跃事务的ReadView的列表情况 **m_ids：[888]** ，因此查询语句会根据当前版本链中小于 **m_ids** 中的最大的版本数据，即查询到的是 Messi。

时间点 T11 情况下的 SELECT 语句：

当前时间点的版本链信息：

![image-20201210173604265](https://jameslin23.gitee.io/2020/12/10/mysql事务/image-20201210173604265.png)

​      此时 SELECT 语句执行，当前数据的版本链如上，因为当前的事务777和事务888 都已经提交，所以此时的活跃事务的ReadView的列表为空 ，因此查询语句会直接查询当前数据库最新数据，即查询到的是 Dybala。

**总结：** **使用READ COMMITTED隔离级别的事务在每次查询开始时都会生成一个独立的 ReadView。**

######  REPEATABLE READ 隔离级别下的ReadView

**在事务开始后读取第一次读取数据时生成一个ReadView（m_ids列表）**

####  MVCC总结：

所谓的MVCC（Multi-Version Concurrency Control ，多版本并发控制）指的就是在使用 **READ COMMITTD** 、**REPEATABLE READ** 这两种隔离级别的事务在执行普通的 SEELCT 操作时访问记录的版本链的过程，这样子可以使不同事务的 `读-写` 、 `写-读` 操作并发执行，从而提升系统性能。

在 MySQL 中， READ COMMITTED 和 REPEATABLE READ 隔离级别的的一个非常大的区别就是它们生成 ReadView 的时机不同。在 READ COMMITTED 中每次查询都会生成一个实时的 ReadView，做到保证每次提交后的数据是处于当前的可见状态。而 REPEATABLE READ 中，在当前事务第一次查询时生成当前的 ReadView，并且当前的 ReadView 会一直沿用到当前事务提交，以此来保证可重复读（REPEATABLE READ）。

## 事务日志

###  redo log

redo log叫做重做日志，是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log）,前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都会存到该日志中。假设有个表叫做tb1(id,username) 现在要插入数据（3，ceshi）

![image-20201210174515027](https://jameslin23.gitee.io/2020/12/10/mysql事务/image-20201210174515027.png)

```mysql
start transaction;
select balance from bank where name="zhangsan";
// 生成 重做日志 balance=600
update bank set balance = balance - 400; 
// 生成 重做日志 amount=400
update finance set amount = amount + 400;
```

![image-20201210174624187](https://jameslin23.gitee.io/2020/12/10/mysql事务/image-20201210174624187.png)

**作用:**

mysql 为了提升性能不会把每次的修改都实时同步到磁盘，而是会先存到Boffer Pool(缓冲池)里头，把这个当作缓存来用。然后使用后台线程去做缓冲池和磁盘之间的同步。

那么问题来了，如果还没来的同步的时候宕机或断电了怎么办？还没来得及执行上面图中红色的操作。这样会导致丢部分已提交事务的修改信息！

所以引入了redo log来记录已成功提交事务的修改信息，并且会把redo log持久化到磁盘，系统重启之后在读取redo log恢复最新数据。

**总结：redo log是用来恢复数据的，用于保障，已提交事务的持久化特性（记录了已经提交的操作）**

###  undo log?

undo log 叫做回滚日志，用于记录数据被修改前的信息。他正好跟前面所说的重做日志所记录的相反，重做日志记录数据被修改后的信息。undo log主要记录的是数据的逻辑变化，为了在发生错误时回滚之前的操作，需要将之前的操作都记录下来，然后在发生错误时才可以回滚。
 还用上面那两张表

![image-20201210175554317](https://jameslin23.gitee.io/2020/12/10/mysql事务/image-20201210175554317.png)

每次写入数据或者修改数据之前都会把修改前的信息记录到 undo log。

**作用**:

undo log 记录事务修改之前版本的数据信息，因此假如由于系统错误或者rollback操作而回滚的话可以根据undo log的信息来进行回滚到没被修改前的状态。

**总结:undo log是用来回滚数据的用于保障，未提交事务的原子性**	



