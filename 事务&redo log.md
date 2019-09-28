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

Checkpoint

