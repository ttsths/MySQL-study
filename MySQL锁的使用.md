# 1.锁的使用



## 1.1 锁的副作用

### 1.1.1 锁等待

```
#正在执行的事务
SELECT * from information_schema.INNODB_TRX;
#当前出现的锁等待
SELECT * from information_schema.INNODB_LOCK_WAITS;
#出现锁等待的锁的详细信息
SELECT * from information_schema.INNODB_LOCKS;
#查看全部线程，辅助定位客户端的主机ip，连接用户名等
show full processlist;
#如果活跃事务少，会显示当前活跃的事务详细信息，多的话只显示概要；最近一次死锁的信息
show engine innodb status;
```

### 1.1.2 死锁

死锁都是由加锁顺序不一致导致的，最常见的update死锁，insert也能造成死锁，有兴趣的可以自行了解

| TransactionA                                                 | TransactionB                                                 |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| start transaction;                                           | start transaction;                                           |
| update t_test_01 set name="name8" where status=8;#持有status=8的锁 |                                                              |
|                                                              | update t_test_01 set name="name9" where status=9;#持有status=9的锁 |
| update t_test_01 set name="name9" where status=9;#等待status=9的锁 |                                                              |
|                                                              | update t_test_01 set name="name8" where status=8;#等待status=8的锁 |
| commit                                                       | Commit                                                       |

```
start TRANSACTION;
update t_test_01 set name="name8" where status=8;
select sleep(10);
update t_test_01 set name="name9" where status=9;
commit;
start transaction;
update t_test_01 set name="name9" where status=9;
update t_test_01 set name="name8" where status=8
commit;
```

### 1.1.3 减少死锁锁等待

1.小事务

事务加锁范围不宜过大，如果比较大，业务上能分割的尽量分割。

例如：订单定时完成的批量，查出一批需要完成的订单，每个订单单独的事务处理，而不是放到一个大的事务里。

2.统一加锁顺序

3.update对应的查询走索引

## 1.2 悲观锁

### 1.2.1 一个例子

```
/**
 * 订单退款
 * @param orderId
 */
public void refund(Long orderId) {
   //select * from order where id={orderId};
    Order order = orderMapper.get(orderId);
    if (order.getStatus() == "已付款") {
        //第三方退款
        thirdPartyRefund();
        order.setStatus("已退款");
        orderMapper.update(order);
   } else {
        throw new Exception();
   }
}
```

有什么问题？

如果是客服给客人退款，不小心点了两次，或者退款比较慢点完又点了一次，如果退款走的是转账……

最简单的解决方案：

```
/**
 * 订单退款
 * @param orderId
 */
@Transactional
public void refund(Long orderId) {
   //select * from order where id={orderId} for update;
    Order order = orderMapper.getAndLock(orderId);
    if (order.getStatus() == "已付款") {
        //第三方退款
        thirdPartyRefund();
        order.setStatus("已退款");
        orderMapper.update(order);
   } else {
        throw new Exception();
   }
}
```

即使同一个订单退款同时出发了两次，由于X锁，第二次请求会阻塞。

这就是悲观锁：假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作。

### 1.2.2 使用场景

1.存在需要控制并发的场景。

2.加锁的对象并发量不大。例如：对一个订单来说，并发主要来自用户对这个订单的操作，量并不大。

3.加锁的范围不能太大。

两方面：

数据库层面：建议只对主键加锁，例如：如果我对order里的userId加锁，影响范围就比较大了。

业务层面：如果订单系统由用户表（user），对用户表里的主键加锁对业务的影响。

## 1.3 乐观锁

### 1.3.1例子

下单减库存

t_stock

条数 400W

| 列名    | 类型    | 说明                   |
| :------ | :------ | :--------------------- |
| id      | bigint  | 产品id主键 primary key |
| amount  | int     | 库存数                 |
| version | integer | 版本                   |

```
@Transactional
public int decAmount(long id) {
    boolean updateFail = true;

    int i = 0;
    for (; i<=5 ; i++) {
      //select * from t_stock where id={id}
        StockEntity stock = stockMapper.get(id);
        stock.setAmount(stock.getAmount() - 1);
      //update t_stock set amount={stock.amount} where id={stock.id} and version = {stock.version};
        int affectCount = stockMapper.update(stock);
        if (affectCount > 0) {
            logger.info("i : {}", i);
            updateFail = false;
            break;
       }
   }
    if (updateFail) {
        logger.error("i: {}", i );
   }
    return i;
}
```

思想是CAS（Compare and Swap），JUC下面的Atomic包利用的就是CPU的CAS操作。

我就在不同隔离级别，不同的索引类型下做了试验：

Jmeter 1秒内1000并发，记录i值。i表示经过了几次CAS操作，0表示1次，1表示2次，以此类推，6表示超过定义的最大循环次数更新失败退出了。

> 注意：全局修改隔离级别后需要重启应用，否则连接池里的连接还是用的修改前的隔离级别，或者直接在连接参数里修改隔离级别。

### 1.3.2 RC级别下高并发结果

Jmeter 1S 1000个的并发

1. id主键索引

```
{0=1, 1=999, 2=0, 3=0, 4=0, 5=0, 6=0}
```

1. idx_id_version普通索引

```
{0=1, 1=999, 2=0, 3=0, 4=0, 5=0, 6=0}
```

1. primary_id_version主键索引

```
{0=64, 1=61, 2=57, 3=63, 4=41, 5=47, 6=667}
```

1. id唯一索引

```
{0=1, 1=999, 2=0, 3=0, 4=0, 5=0, 6=0}
```

1. id普通非唯一索引

```
{0=31, 1=40, 2=47, 3=23, 4=28, 5=21, 6=810}
```

#### 1.结果

期望的结果更新次数在1～6之间都有分布的，RC级别下只有id和version是主键索引，或者id是非唯一的普通索引的时候才符合预期。其他情况除了一条是1次更新成功，其他都是第二次更新成功。

#### 2.分析

id主键索引/id version 联合索引 /id唯一索引的情况下，第一次循环里的`update t_stock set amount={stock.amount} where id={stock.id} and version = {stock.version};`，即使version不对也会对记录加锁，1000个请求过来，只有一个请求获取了锁并更新成功，其他锁加入等待队列；等到第一个请求更新成功，后面某个获取到锁的请求必然在第一个循环里更新失败，但并不会释放锁，第二次循环会更新成功。

primary_id_version是联合主键的时候`update t_stock set amount={stock.amount} where id={stock.id} and version = {stock.version};`锁主键的时候如果version不对主键并不存在，所以不会锁记录。

id是普通非唯一索引的时候`update t_stock set amount={stock.amount} where id={stock.id} and version = {stock.version};`会锁[stock.id](http://stock.id/)对应的记录，单发现不符合where条件里的version会立即释放锁，参考`2.4.2 RC级别下update … where 加锁后释放锁`里的第二条。

### 1.3.3RR级别下高并发结果

1. id主键索引

```
{0=11, 1=0, 2=0, 3=0, 4=0, 5=0, 6=989}
```

1. id主键索引+idx_id_version普通索引

```
{0=13, 1=0, 2=0, 3=0, 4=0, 5=0, 6=987}
```

1. primary_id_version主键索引

```
{0=304, 1=0, 2=0, 3=0, 4=0, 5=0, 6=696}
```

1. id 唯一索引

```
{0=13, 1=0, 2=0, 3=0, 4=0, 5=0, 6=987}
```

#### 1.结果

如果第一次没有更新成功，后面就不会更新成功

#### 2.分析

RR级别下MVCC的一致读导致第一次循环如果没有更新成功，即使加了锁，第二次的快照读的结果和第一次还是一样，这样获取的version还是第一次的version，后面的更新都不会更新上。

### 1.3.4乐观锁的变体

```
@Transactional
public void decAmount(long id) {
  //update t_stock set amount=amount-1 where id={stock.id} and amount >= 1;
  int affectCount = stockMapper.decAmount(stock);
  if (affectCount > 0) {
    //扣减成功
 } else {
    //扣减失败
 }
}
```

利用amount替换version，没有仿CAS的操作，其实也不算乐观锁了。

### 1.3.5总结

MySQL底层用锁实现的，想实现无锁化的乐观锁并不现实，使用起来也有坑，并不推荐使用。

## 1.4 分布式锁

### 1.4.1Redis实现的分布式锁

redis中有个命令`setNX`，是一种CAS操作，不同于一般的`set`命令直接覆盖原值，`setNx`在更新的时候会判断当前`key`是否存在，如果存在返回`false`，如果不存在设置`value`并返回`true`。

下面的代码利用这个CAS操作写简单的乐观锁：

```
/**
 * 获取锁
 *
 * @param key 锁id
 * @return 锁结果
 */
public boolean tryLock(String key) {
        try {
        #setNx
            if (redisTemplate.opsForValue().setIfAbsent(key, "")) {
               redisTemplate.expire(key, 5000, TimeUnit.MILLISECONDS);
               return true;
           }
         }
       } catch (Exception e) {
            LOGGER.error("get lock {} error", key, e);
       }
        return false;
   }
```

上面只是简单演示基本原理，实际使用中需要考虑很多问题。如redis失效和expire失败导致锁不释放（redis 2.8版本支持setnx命令支持设置失效时长；reids是单线程，也可以用eval执行lua脚本的方式实现）。

redis方案的分布式锁推荐RedissonLock，实现java.util.concurrent.Lock接口，用起来很方便；内部用redis的eval命令执行lua脚本，可以参看<https://github.com/angryz/my-blog/issues/4>。

### 1.4.2 使用分布式锁

扣减库存的例子

```
@Transactional
public void decAmount(long id) {
    Lock redissonLock = redissonClient.getLock(id);
    if(redissonLock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            //select * from t_stock where id={id}
            StockEntity stock = stockMapper.get(id);
            stock.setAmount(stock.getAmount() - 1);
            //update t_stock set amount={stock.amount} where id={stock.id};
            stockMapper.update(stock);
       } catch (Exception e) {
            //todo
       } finally {
            redissonLock.unlock();
       }
   }
}
```

存在的问题：强依赖redis，如果redis挂了怎么办。

修改：

```
@Transactional
public void decAmount(long id) {
    Lock redissonLock = redissonClient.getLock(id);
    if(boolean lockSuccess = redissonLock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            //select * from t_stock where id={id}
            StockEntity stock = stockMapper.get(id);
            stock.setAmount(stock.getAmount() - 1);
            //update t_stock set amount={stock.amount} where id={stock.id} and version = {stock.version};
            int size = stockMapper.updateByVersion(stock);
            boolean success = size > 0;
       } catch (Exception e) {
            //todo
       } finally {
            if (lockSuccess) {
                redissonLock.unlock();
           }
       }
   }
}
```

#### 1. Redis锁+数据库锁双重保障的方式

#### 2. 相比只有数据库加锁的优点

1.redis锁生效的时候，数据库没有锁等待。

2.redis失效的时候

可以考虑服务降级：例如上面的乐观锁，去掉循环之后，更新一次如果失败就返回失败；

或者服务不降级：用数据库锁扛着。

#### 3. 相比只用redis加锁的优点

1.不强依赖redis

### 1.4.3 使用场景

1.只想在特定的操作加锁

例如：同一用户每次只允许下一单，如果用数据库锁，可能会锁住用户相关的所有操作；这时候用分布式锁没有问题，因为锁对象（redis里的key值）定义很自由。用户退款可以定义为：`lock:user:refund:{userId}`,用户下单可以定义为：`lock:user:order:{userid}`

2.锁对象的并发量很大

高并发的时候如果使用数据库锁，会有很长的锁等待队列，数据库连接也被占；虽然锁等待超时会抛异常，放弃等待，等待时间也很难控制。

经典场景：秒杀，对单个产品对象的并发。

3.考虑好锁失效的场景和处理方案