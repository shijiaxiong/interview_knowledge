### 缓存异常

#### 缓存雪崩

**缓存雪崩**是指缓存同一时间大面积的失效，所以，后面的请求都会落到数据库上，造成数据库短时间内承受大量请求而崩掉。

##### 缓存雪崩的事前事中事后的解决方案如下。

- 事前：redis 高可用，主从+哨兵，redis cluster，缓存设置随机过期时间避免全盘崩溃。
- 事中：本地内存缓存(Java有ehcache缓存) + 限流&降级，避免 MySQL 被打死。
- 事后：redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

限流组件，可以设置每秒的请求，有多少能通过组件，剩余的未通过的请求，怎么办？**走降级**！可以返回一些默认的值，或者友情提示，或者空白的值。

好处：

- 数据库绝对不会死，限流组件确保了每秒只有多少个请求能通过。
- 只要数据库不死，就是说，对用户来说，2/5 的请求都是可以被处理的。
- 只要有 2/5 的请求可以被处理，就意味着你的系统没死，对用户来说，可能就是点击几次刷不出来页面，但是多点几次，就可以刷出来一次。

#### 缓存穿透

**缓存击穿**是指缓存中没有但数据库中有的数据（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力。和缓存雪崩不同的是，缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库。

**解决方案**

1. 设置热点数据永远不过期。
2. 加互斥锁，基于 redis or zookeeper 实现互斥锁，等待第一个请求构建完缓存之后，再释放锁，进而其它请求才能通过该 key 访问数据。

#### 缓存击穿

**缓存穿透**是指缓存和数据库中都没有的数据(无论访问多少次都无法建立缓存)，导致所有的请求都落到数据库上，造成数据库短时间内承受大量请求而崩掉。

**解决方案**

1. 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截。
2. 写一个空值到缓存里去。缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击。
3. 采用布隆过滤器，将所有可能存在的数据哈希到一个足够大的 bitmap 中(预先对数据库中的数据做加载)，一个一定不存在的数据会被这个 bitmap 拦截掉，从而避免了对底层存储系统的查询压力。

### 过期键的删除策略

#### Redis的过期键的删除策略

过期策略通常有以下三种：

- 定时过期：每个设置过期时间的key都需要创建一个`定时器`，到过期时间就会立即清除。该策略可以立即清除过期的数据，对内存很友好；但是会占用大量的CPU资源去处理过期的数据，从而影响缓存的响应时间和吞吐量。
- 惰性过期：`访问时判断是否已过期`，过期则清除。该策略可以最大化地节省CPU资源，却对内存非常不友好。极端情况可能出现大量的过期key没有再次被访问，从而不会被清除，占用大量内存。
- 定期过期：每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。该策略是前两者的一个折中方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得CPU和内存资源达到最优的平衡效果。 (expires字典会保存所有设置了过期时间的key的过期时间数据，其中，key是指向键空间中的某个键的指针，value是该键的毫秒精度的UNIX时间戳表示的过期时间。键空间是指该Redis集群中保存的所有键。)

Redis中同时使用了`惰性过期`和`定期过期`两种过期策略。

### 内存相关

#### 内存淘汰策略有哪些

Redis的内存淘汰策略是指在Redis的用于缓存的内存不足时，怎么处理需要新写入且需要申请额外空间的数据。

>  最近最少使用、随机删除、

- noeviction：当内存不足以容纳新写入数据时，新写入操作会报错。
- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。
- allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。
- volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。
- volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。
- volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。

### 事务

#### Redis事务的三个阶段

1. 事务开始 MULTI
2. 命令入队
3. 事务执行 EXEC

事务执行过程中，如果服务端收到有EXEC、DISCARD、WATCH、MULTI之外的请求，将会把请求放入队列中排队

#### Redis事务相关命令

Redis事务功能是通过MULTI、EXEC、DISCARD和WATCH 四个原语实现的

Redis会将一个事务中的所有命令序列化，然后按顺序执行。

1. **redis 不支持回滚**，“Redis 在事务失败时不进行回滚，而是继续执行余下的命令”， 所以 Redis 的内部可以保持简单且快速。
2. **如果在一个事务中的命令出现错误，那么所有的命令都不会执行**；
3. **如果在一个事务中出现运行错误，那么正确的命令会被执行**。

- WATCH 命令是一个乐观锁，可以为 Redis 事务提供 check-and-set （CAS）行为。 可以监控一个或多个键，一旦其中有一个键被修改（或删除），之后的事务就不会执行，监控一直持续到EXEC命令。
- MULTI命令用于开启一个事务，它总是返回OK。 MULTI执行之后，客户端可以继续向服务器发送任意多条命令，这些命令不会立即被执行，而是被放到一个队列中，当EXEC命令被调用时，所有队列中的命令才会被执行。
- EXEC：执行所有事务块内的命令。返回事务块内所有命令的返回值，按命令执行的先后顺序排列。 当操作被打断时，返回空值 nil 。
- 通过调用DISCARD，客户端可以清空事务队列，并放弃执行事务， 并且客户端会从事务状态中退出。
- UNWATCH命令可以取消watch对所有key的监控。

#### Redis事务保证原子性吗，支持回滚吗

Redis中，单条命令是原子性执行的，但**事务不保证原子性，且没有回滚**。事务中任意命令执行失败，其余的命令仍会被执行。

### 集群方案

#### Redis哈希槽概念

Redis集群没有使用一致性hash,而是引入了哈希槽的概念，Redis集群有16384个哈希槽，每个key通过CRC16校验后对16384取模来决定放置哪个槽，集群的每个节点负责一部分hash槽。

在redis节点发送心跳包时需要把所有的槽放到这个心跳包里，以便让节点知道当前集群信息，16384=16k，在发送心跳包时使用char进行bitmap压缩后是2k（2 * 8 (8 bit) * 1024(1k) = 16K），也就是说使用2k的空间创建了16k的槽数。

详解：[为什么Redis集群有16384个槽](https://www.cnblogs.com/rjzheng/p/11430592.html)

#### Redis集群最大节点个数是多少

16384

### 分布式问题



### 数据类型实现原理及应用

聚合计算
set
统计每天新增和留存
记录当天访问用户
记录历史访问用户
sdiffstore 差集是新增
sinterstore 交集是留存

排序统计
list 队列
sort-list  排行榜

二值状态统计
bitmap

基数统计
HyperLogLog

附近门店、打车

Geo 对经纬度做二分区操作生成二进制码，二进制码合并(偶数经度编码值、奇数纬度编码值)后做sort set的权重分数。



###  并发访问

29

30

31

36

39



### 其他问题

[Redis面试题](https://cloud.tencent.com/developer/article/1612347)

#### Redis与Memcached的区别

[redis和memcached的区别](https://github.com/shishan100/Java-Interview-Advanced/blob/master/docs/high-concurrency/redis-single-thread-model.md)

#### 如何保证缓存与数据库双写时的数据一致性？

[如何保证数据库与缓存的双写一致](https://github.com/shishan100/Java-Interview-Advanced/blob/master/docs/high-concurrency/redis-consistence.md)

#### 布隆过滤器

##### 简介

布隆过滤器（Bloom Filter）是一个基于hash的概率性的数据结构，它实际上是一个很长的二进制向量，可以检查一个元素可能存在集合中，和一定不存在集合中。它的优点是空间效率高，但是有一定false positive(元素不在集合中，但是布隆过滤器显示在集合中)。

##### 原理

布隆过滤器就是一个长度为`m`个bit的bit数组，初始的时候每个bit都是0，另外还有`k`个hash函数。

当加入一个元素时，先用`k`个hash函数得到`k`个hash值，将`k`个hash值与bit数组长度取模得到个`k`个位置，将这`k`个位置对应的bit置位1。

参数设置参考

[白话布隆过滤器](https://blog.huoding.com/2020/06/22/825)

##### 实现

hash函数的选择，murmur3、FNV

[Redis实现布隆过滤器](https://learnku.com/articles/46442)