# Explain 详解

```mysql 
mysql> EXPLAIN SELECT 1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.01 sec)
```

| 列名          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| id            | 在一个大的查询语句中每个select关键字都对应一个唯一ID         |
| Select_type   | select关键字对应的查询类型                                   |
| table         | 表名                                                         |
| partitions    | 匹配的分区信息                                               |
| type          | 针对单标访问的方法<br />system`，`const`，`eq_ref`，`ref`，`fulltext`，`ref_or_null`，`index_merge`，`unique_subquery`，`index_subquery`，`range`，`index`，`ALL |
| possible_keys | 可能用到的索引                                               |
| key           | 实际使用到的索引                                             |
| Key_len       | 实际使用到的索引长度                                         |
| ref           | 当使用索引等值查询时，与索引列进行等值匹配的对象信息         |
| Rows          | 预估的需要读取的记录条数                                     |
| filtered      | 某个表经过搜索条件过滤后剩余记录条数的百分比                 |
| Extra         | 一些额外的信息                                               |

explain每个表对应一条记录，连接查询时出现在前面的是驱动表，后面的是被驱动表。

查询优化器可能对涉及子查询的语句进行重写，转换为连接查询。

- 查询中包含子查询

- 查询中包含union

  ```mysql
  mysql> EXPLAIN SELECT * FROM s1  UNION SELECT * FROM s2;
  +----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
  | id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
  +----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
  |  1 | PRIMARY      | s1         | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |   100.00 | NULL            |
  |  2 | UNION        | s2         | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9954 |   100.00 | NULL            |
  | NULL | UNION RESULT | <union1,2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | Using temporary |
  +----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
  3 rows in set, 1 warning (0.00 sec)
  ```

  union需要去重，所以多了第三条。union all则不需要为结果集去重。

  

