[TOC]



# MVCC

## 事务隔离级别

事务并发执行的问题

- Dirty Write 

  一个事务修改了另一个事务未提交事务修改过的数据

- Dirty Read

  一个事务读到了另一个未提交的事务修改过的数据

- Non-Repeatable Read

  一个事务只能读到另一个已经提交事务修改过的数据，并且其他事务每次对该数据进行修改并提交后，该事务都能查询到最新的值

- Phantom

  一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次查询时，能把另一个事务插入的记录页读出来。

### SQL标准中的四种隔离级别

- `READ UNCOMMITTED`：未提交读
- `READ COMMITTED`：已提交读
- `REPEATABLE READ`：可重复读。
- `SERIALIZABLE`：可串行化。

>Oracle只支持READ COMMITED和SERIALIZABLE隔离级别。
>
>MySQL在RR隔离级别下也可以禁止幻读问题的发生

## MVCC原理

### 版本链

InnoDB中，聚簇索引记录中都包含两个必要的隐藏列。

- trx_id：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的事务ID赋给trx_id
- Roll_pointer:每次对某条聚簇索引记录改动时，会把旧版本写入到undo日志，这个相当于指针，可以通过它来找到修改前该记录的信息

对一条记录的多次修改，所有版本都会被`roll_pointer`属性连接成一个链条，版本链的头结点就是当前记录的最新值

### ReadView

- `m_ids`:生成ReadView时，当前系统中活跃的读写事务的`事务id`列表

- `min_trx_id`:生成ReadView时，当前系统中活跃的读写事务中最小的事务ID。

- `max_trx_id`：生成ReadView时，系统中应该分配给下一个事务的id值

  >小贴士： 注意max_trx_id并不是m_ids中的最大值，事务id是递增分配的。比方说现在有id为1，2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生成ReadView时，m_ids就包括1和2，min_trx_id的值就是1，max_trx_id的值就是4。

- `creator_trx_id`:表示生成该ReadView的事务的事务id

:warning:**可见性的判断标准**

- `trx_id`== `creator_trx_id` :当前事务在访问它修改过的记录
- `trx_id` < `min_trx_id`:该版本对当前事务可见
- `Trx_id` > `max_trx_id`:该版本对当前事务不可见
- `min_trx_id` <= `trx_id` <= `max_trx_id`  && `trx_id` not in `m_ids`:该版本对于当前事务可见
- `min_trx_id` <= `trx_id` <= `max_trx_id`  && `trx_id` in `m_ids`:该版本对于当前事务不可见

在`MySQL`中，`READ COMMITTED`和`REPEATABLE READ`隔离级别的的一个非常大的区别就是它们生成ReadView的时机不同。

**READ COMMITED —每次读取数据前都生成一个ReadView**

**RR —在第一次读取数据时生成一个ReadView**

:imp:MVCC 小结：

多版本并发控制指的是使用`RC`，`RR`两个隔离级别下，不同事务的`读-写`，`写-读`操作并发执行，提升系统性能。`RC`和`RR`最大的不同在于生成ReadView的时机，Read Commited每次select都会生成一个新的ReadView，而Repeatable read 只是在第一次进行select操作时生成ReadView。



### 关于purge

用于删除`update undo日志`