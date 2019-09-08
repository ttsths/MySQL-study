---
typora-root-url: ../blog
---

# MySQL （2-9）

## MySQL安装的重要信息

mysql/bin

启动MySQL

./bin/mysqld

客户端

mysql -h 主机名 -u  用户名 -p 密码

mysql -hlocalhost -uroot -ptt123

## 客户端SQL的请求过程

### 1，连接管理

### 2，解析与优化

- 查询缓存（MySQL8.0中删除）
- 语法解析
- 查询优化

> 可以还用EXPLAIN 查看语句的执行计划

### 3，存储引擎

只需关注InnoDB与MyISAM Memory

```mysql
## 查看服务器支持的存储引擎
SHOW ENGINES
```

创建表的时候需要制定存储引起

```mysql
CREATE TABLE 表名(
		建表语句；
) ENGINE = 存储引擎名称；
## 修改
ALTER TABLE 表名 ENGINE = 存储引擎名称；
```

## 字符集

- ASCII
- ISO 8859-1
- GB2312
- GBK
- utf8

> MySQL中的utf8(1~3字符)和utf8mb4 (1~4字符)
>
> utf8字符集表示一个字符需要使用1~4个字节
>
> Utf8mb4：正宗的utf8字符集
>
> MySQL有4个级别的字符集和比较规则：服务器级别，数据库级别，表级别，列级别

```mysql
## character 字符集：某个字符范围的编码规则  
## collate  比较规则：针对某个字符集中的字符比较大小的一种规则
create database testdb character set utf8 collate utf8_unicode_ci;
```

### InnoDB

> InnoDB是一种将表中的数据存储到磁盘上的存储引擎，先将数据在内存中处理，再讲数据刷新到磁盘中。InnoDB讲数据划分为若干个页，以页作为磁盘和内存之间交互的基本单位，InnoDB中页的大小一般为16KB。

- 行格式

Compact

额外信息：变长字段长度列表、NULL值列表、记录头信息

变长字段长度列表：VARCHAR(M)、VARBINARY(M)、TEXT类型、BLOB类型。将所有变长字段的真实数据占用的字节长度都存放在记录的开头部位，形成变长字段长度列表，各变长字段数据占用的字节数按照<b>逆序存放</b>。存储<font color="red"><b>非NULL的列</b></font>内容占用的长度

NULL值列表：不允许为NULL的列不存储，例如主键。如果表中没有允许存储NULL的列，则NULL值列表就不存在。<b>逆序排序</b>,16进制表示

记录头信息：5个字节，40个二进制位

| 名称         | 大小（bit） | 描述                                                         |
| ------------ | ----------- | ------------------------------------------------------------ |
| 预留位1      | 1           | 无使用                                                       |
| 预留位2      | 1           | 无使用                                                       |
| delete_mask  | 1           | 标记该记录是否被删除                                         |
| min_rec_mask | 1           | B+树的每层非叶子节点中的最小记录都会添加该记录               |
| n_owned      | 4           | 表示当前记录拥有的记录数                                     |
| heap_no      | 13          | 表示当前记录堆的位置信息                                     |
| record_type  | 3           | 记录类型；0：普通记录，1：B+树非叶子节点记录，2：最小记录，3：最大记录 |
| next_record  | 16          | 下一条记录的相对位置                                         |

真实信息：列1的值、列2的值、列N的值

隐藏列：

| **列名**       | **是否必须** | **占用空间** | 描述                   |
| -------------- | :----------: | ------------ | ---------------------- |
| row_id         |      否      | 6字节        | 行ID，唯一标识一条记录 |
| transaction_id |      时      | 6字节        | 事务ID                 |
| roll_pointer   |      是      | 7字节        | 回滚指针               |



Redundant

MySQL5.0之前的一种行格式

Dynamic

5.7默认的行格式，类似于Compact行格式，不同在于处理`行溢出`数据时，不会记录真实数据处存储真实数据的部分信息，而是将所有字节都存储到其他页面，只记录溢出页的地址

Compressed

与Dynamic不同在于会采用压缩算法进行压缩节省空间

```mysql
CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称
    
ALTER TABLE 表名 ROW_FORMAT=行格式名称
```

<b>总结</b>

1，页是MySQL中磁盘和内存交互的基本单位，也是MySQL管理存储空间的基本单位

2，一般一页存储16KB，当记录中的数据太多，会`行溢出`

### 数据页结构

页有多种类型：Insert Buffer信息页，INODE信息页，INDEX页等等

数据页的结构

|        名称        |       中文名       | 占用空间大小（字节） |         简单描述         |
| :----------------: | :----------------: | :------------------: | :----------------------: |
|    File Header     |      文件头部      |        38字节        |     页的一些通用信息     |
|    Page Header     |      页面头部      |        56字节        |   数据页专业的一些信息   |
| Infimum + Supermum | 最小记录和最大记录 |        26字节        |     两个虚拟的行记录     |
|    User Records    |      用户记录      |        不确定        |   实际存储的行记录内容   |
|     Free Space     |      空闲空间      |        不确定        |    页中尚未使用的空间    |
|   Page Directory   |      页面目录      |        不确定        | 页中的某些记录的相对位置 |
|    File Trailer    |      文件尾部      |        8字节         |      校验页是否完整      |

垃圾链表：被删除的记录组成，这个链表占用的空间称为可重用空间



**Page Directory(页目录)**

记录页中按照**主键**值由小到大串联成<font color='red'><b>单向链表</b></font>，删除的记录不会会做标识，组成垃圾链表.会自动插入最小记录Infimum Supermum(虚记录，但是也在链表内)。



将页中的数据分组，将组内最后一条记录的头信息中的`n_owned`属性表示该组拥有多少记录。

每个组的最后一条记录的地址偏移量(Slot)单独取出存储在靠近页的尾部的地方，即Page Directory,

分组中的记录条数规则：对于最小记录所在的分组只能有 ***1***条记录，最大记录所在的分组拥有的记录条数只能在 ***1~8*** 条之间，剩下的分组中记录的条数范围只能在是 ***4~8*** 条之间。

<font color='red'><b>二分法查找</b></font>:5个Slot:0,1,2,3,4 中查找id=6的记录 

low =0 high=4

(0+4)/2 = 2 —>  Slot2 最大的记录为8，8>6  —》 high=2

(0+2)/2 = 1 —> Slot1 最大的记录为4， 4<6 —> low = 1

High-low =1  —>确定6在Slot2，通过Slot1的最后一条记录找到Slot2，遍历Slot2

 **Page Header**

PAGE_N_DIR_SLOTS				槽数量

PAGE_HEAP_TOP					未使用的空间最小地址

PAGE_N_HEAP						本页中记录的数量(包括最大，最小和删除的记录)

PAGE_FREE							   第一个已经被标记为删除的记录地址

PAGE_GARBAGE						已删除记录占用的字节数

PAGE_LAST_INSERT				最后插入记录的位置

PAGE_DIRECTION					记录插入的方向

PAGE_N_DIRECTION				一个方向连续插入的记录数

PAGE_N_RECS							该页中记录的数量(不包括最大，最小和删除的记录)

PAGE_MAX_TRX_ID				当前页的最大事务ID，仅在二级索引中定义	

PAGE_LEVEL							当前页在B+树中所处的层级

PAGE_INDEX_ID					索引ID：表示当前页属于哪个索引

PAGE_BTR_SEG_LEAF			B+树叶子段的头部信息，仅在B+树的Root页定义

PAGE_BTR_SEG_TOP			B+树非叶子段的头部信息，仅在B+树的Root页定义



**File Header**

| 名称                             | 占用空间大小 | 描述                                                         |
| -------------------------------- | ------------ | ------------------------------------------------------------ |
| FIL_PAGE_SPACE_OR_CHKSUM         | 4            | 页的校验和（checksum值）                                     |
| FIL_PAGE_OFFSET                  | 4            | 页号                                                         |
| FIL_PAGE_PREV                    | 4            | 上一个页的页号                                               |
| FIL_PAGE_NEXT                    | 4            | 下一个页的页号                                               |
| FIL_PAGE_LSN                     | 8            | 页面被最后修改时对应的日志序列位置（英文名是：Log Sequence Number） |
| FIL_PAGE_TYPE                    | 2            | 该页的类型                                                   |
| FIL_PAGE_FILE_FLUSH_LSN          | 8            | 仅在系统表空间的一个页中定义，代表文件至少被刷新到了对应的LSN值 |
| FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID | 4            | 页属于哪个表空间                                             |

**File Trailer**

因为存储是内存然后刷新到磁盘，所以为了防止内存同步到磁盘意外



## B+树索引

**页分裂**：下一个数据页中用户记录的主键值必须大于上一个页中用户记录的主键值。（如果不符合则会将记录移动）（<font color=red>性能损耗</font>）

目录项记录：存放目录项（pageNo,key:数据页中最小的主键值），`record_type`值是1，普通记录的值是0。

当表中数据非常多会产生更高级的目录，目录项记录页产生多个层级。

### 聚簇索引

B+树的最底层的节点：存放实际用户记录。—》叶子节点

页内的记录是按照主键大小顺序排成一个单向链表

数据页之间是根据页中用户记录的主键大小排成一个双向列表

存放目录项的页有不同的层次，同一个层中的页根据页中目录项记录的主键大小排成一个双向链表

在`InnoDB`存储引擎中，`聚簇索引`就是数据存储的方式，所有数据存储在叶子节点。

聚簇索引并不需要显示使用INDEX语句创建

### 二级索引

不同于聚簇索引：

- 使用列的值大小进制记录和页的排序：
  - 页内的记录是按照指定列值大小排成一个单向链表
  - 各个存放用户记录的页也是根据页中记录的指定列大小排成一个双向链表
  - 存放目录项记录的页分为不同的层次，在同一层次中的页也是根据页中目录项记录的指定列大小顺序排成一个双向链表。
- B+树的叶子节点存储的不是完整的用户记录，而只是**指定列+主键**的值
- 目录项记录中不再是**主键+页号**的搭配而是**指定列+页号+主键值**

<font color=red>回表</font>：在二级索引中查到主键值后，需要在聚簇索引中再查一遍的过程

## 联合索引

每条`目录项记录`都由`c2`、`c3`、`页号`三部分组成，各条记录先按照c2列的值进行排序，如果记录的c2列的值相同，则按照c3列的值进行排序。

B+树的叶子节点由c2+c3+主键列组成

**B+树的根节点自诞生起，便不会再移动**

MyISAM存储引擎会将表中的索引信息另外存储到一个索引文件中，存储的是主键值+行号的组合，所以MyISAM在查找的时候需要回表，即MyISAM 中的建议的索引相当于全部都是二级索引

### MySQL中创建和删除索引的语句

```mysql
## 创建索引
CREATE TABLE 表名(
	各种列的信息 ... ,
  [key|INDEX] 索引名(需要被索引的单个列或多个列)
)

## 修改索引
ALTER TABLE 表名 ADD [INDEX | KEY] 索引名 (需要被索引的单列或者多列);

## 删除索引
ALTER TABLE 表名 DROP [INDEX | KEY] 索引名;
```

## B+树索引的使用

- 空间代价
- 时间代价

### B+树索引适用的条件

- 最左匹配
- 所有记录都是按照索引列的值从小到大的顺序排好序的
- like 'As%' 走索引
- 匹配范围值 name > 'Asa' and name <= 'Barlow'
- 精确匹配某一列并范围匹配到另一列
- 用于排序，注意：order by 后面的列的顺序需要按照联合索引中列的顺序给出
- ASC、DESC混用的情况**无法使用索列**
- where子句出现非排序使用到的索引列**无法使用索引**

- 排序列包含非同一索引的列**无法使用索引**
- 排序列使用了复杂的表达式**无法使用索引**

### 回表的代价

性能：顺序IO > 随机IO

查询优化器会将表中的记录计算一些统计数据，然后和需要回表的数据比较。

```mysql 
## 全表扫描
SELECT * FROM person_info ORDER BY name, birthday, phone_number;

## 二级索引+回表
SELECT * FROM person_info ORDER BY name, birthday, phone_number LIMIT 10;
```

### 覆盖索引

查询项只有索引中的列

```mysql
SELECT name, birthday, phone_number FROM person_info WHERE name > 'Asa' AND name < 'Barlow'
```

### 如何挑选索引

- 只为用于搜索、排序或分组的列创建索引

> ​	`where`子句中的列、连接子句中的连接列，或者出现在`order by`、`group by`子句中的列

- 列的基数

> 列的基数：某一列中不重复数据的个数。**<font color=red>在记录行数一定的情况下，列的基础越大，该列中的值越分散，基数越小，列中的值越集中</font>**（PS：一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录。）

- 索引列的类型尽量小
- 索引字符串值得前缀

> utf8字符集一个字符占用1~3个字节。而索引列的字符串前缀其实也是排好序的，—可以的话只对字符串的前几个字符进行索引

```mysql 
# name(10)就是例子
CREATE TABLE person_info(
    name VARCHAR(100) NOT NULL,
    birthday DATE NOT NULL,
    phone_number CHAR(11) NOT NULL,
    country varchar(100) NOT NULL,
    KEY idx_name_birthday_phone_number (name(10), birthday, phone_number)
);   
```

- 让索引列在比较表达式中单独出现

- 主键插入顺序，避免不必要的页分裂，**自增主键的好处**

- 冗余和重复的索引（**禁止**）

## MySQL的数据目录

系统表空间

独立表空间

test.frm  表结构

test.idb 表数据及索引

视图.frm