# InnoDB 的 Buffer Pool

## 缓存的重要性

默认的缓冲池是128M

```
[server]
innodb_buffer_pool_size = 268435456
```

Buffer Pool由控制片(约808字节)，缓存页组成。

free链表(双向链表)记录Buffer Pool中的空闲缓存页

当Buffer Pool中的内存不足时，移除旧的缓存页。

## LRU(Last Recently Used)链表管理

`最近最少使用的原则,将最近一次使用的缓存页至于表头，将LRU链表按照一定比例分成两截，一部分存储使用频率较高的缓存页(热数据|young区域)，一部分存储使用频率不高的数据（冷数据|old区域）；由系统变量innodb_old_blocks_pct的值确定`,

在对某个处在`old`区域的缓存页进行第一次访问时就在它对应的控制块中记录下来这个访问时间，如果后续的访问时间与第一次访问的时间在某个时间间隔内，那么该页面就不会被从old区域移动到young区域的头部，否则将它移动到young区域的头部。上述的这个间隔时间是由系统变量`innodb_old_blocks_time`控制的

`innodb_buffer_pool_instances`指定buffer pool的数量，每个BufferPool实例由若干个chunk组成