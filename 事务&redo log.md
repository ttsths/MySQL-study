[TOC]



# 事务

## 事务的特性

- Atomicity-原子性
  - 事务内的操作要么全部成功，要不全部不成功
- Consistency-一致性
  - 事务提交前后符合数据库的数据时一致
- Isolation-隔离性
  - 事务之间是相互隔离，互不影响
- Durability-持久性
  - 事务一旦提交，无法改变

## 事务的状态

活动的、部分提交的、提交的、失败的、中止的

## 事务的语法

```mysql
## 开启事务
BEGIN;
START TRANSACTION;
## 只读事务
START TRANSACTION READ ONLY;
## 读写事务
START TRANSACTION READ WRITE
## 一致性读
START TRANSACTION WITH CONSISTENT SNAPSHOT
## 提交事务
COMMIT;
## 手动中止事务
ROLLBACK;
## 隐式事务

```

- 隐式事务
  - 当我们使用`ALTER USER`、`CREATE USER`、`DROP USER`、`GRANT`、`RENAME USER`、`REVOKE`、`SET PASSWORD`等语句时也会隐式的提交前边语句所属于的事务。
- 保存点-savepoint



# Redo重做日志

在执行事务的过程中，每执行一条语句，就有可能产生若干条redo日志

## redo日志格式

- type：日志类型
- Space ID：表空间ID
- page number:页号
- data:具体内容

**redo日志会把事务在执行过程中对数据库所做的所有修改都记录下来，在之后系统奔溃重启后可以把事务所做的任何修改都恢复出来。**

## Mini-Transaction

以组的形式写入redo日志

以下情况组不可分割

- 更新`Max Row ID`属性时产生的`redo`日志是不可分割的。
- 向聚簇索引对应`B+`树的页面中插入一条记录时产生的`redo`日志是不可分割的。
- 向某个二级索引对应`B+`树的页面中插入一条记录时产生的`redo`日志是不可分割的。
- 还有其他的一些对页面的访问操作时产生的`redo`日志是不可分割的。。。

## redo日志的写入过程

redo log block

Redo 日志缓冲区512KB的block

redo日志的刷盘时机

## redo日志文件组

Checkpoint：redo日志只是为了系统奔溃后恢复脏页用的，如果对应的脏页已经刷新到磁盘，那么相对应的redo日志就不需要了（通过更新flush链表）checkpoint_lsn增加的形式来标识可以被覆盖的redo日志。

Log Sequeue Number:日志序列号(lsn)

日志首选写入到`log_buffer`中，之后才会被刷新到磁盘上的redo日志文件。通过`flushed_to_disk_lsn`参数来刷新，初始值未`lsn`的初始值**8704**。

## 奔溃恢复

- 确认恢复的起点：redo日志文件组的第一个文件的管理信息中有两个block中选出checkpoint_no比较大的对应的checkpoint_lsn对应的redo日志的checkpoint_offset
- 确认恢复的终点：`log clock header`中的`LOG_BLOCK_HDR_DATA_LEN`的属性，该属性记录了当前block里面使用了多少字节的空间， 未被填满的永远为512
- 如何恢复：
  - 使用哈希表，根据redo日志的`space ID`和`page number`属性计算出散列值（可以算出相同的页），多个相同的散列值(槽)之间使用链表连接