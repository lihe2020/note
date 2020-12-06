#### 1. redis是什么？

redis是一种基于key-value的nosql数据库，它提供了发布订阅、键过期、事务、lua脚本、高可用集群等功能。

应用场景包括：缓存、计数器、分布式session、分布式锁、布隆过滤器、发布订阅

#### 2. 为什么redis会这么快？

总的来说有4个方面的特点导致它的并发非常高、访问速度非常快：

   1. 基于ANSI C开发。采用C语言开发的好处是底层代码执行效率高，依赖性低，而且系统兼容性好，稳定性高。
   2. 纯内存访问。redis将所有数据存放在内存中，内存访问通常很快，大都是纳秒级别，这是redis快的重要基础。
   3. 单线程。单线程简化了算法的实现，并发的数据结构实现不但困难而且也很麻烦。第二，单线程避免了线程切换以及资源竞争带来的性能消耗，对服务端开发来说，锁和上下文切换通常是性能杀手。注意，redis6.0新增了多线程功能，主要用于读取socket、解析请求、回写soket等io耗时操作，核心部分其实还是单线程。
   4. 非阻塞多路复用IO。多路指的是多个网络连接，复用指的是复用同一个线程。采用多路 I/O 复用技术的好处是可以在同一个线程中处理多个 I/O 请求，尽量减少网络 I/O 的消耗，提升使用效率。多路复用说到底就是由内核帮你监听socket fd，有消息到达会通知你去处理。

#### 3. redis有哪些缺点？

1. redis集群架构存在某些问题
2. redis缺少基于角色的权限控制系统，权限设计的不够

#### 4. redis有哪些数据结构？

   - String 用于缓存，计数器等
   - Hash 用于存储键值映射关系，如车型论坛映射
   - List 当存储的数据量小时List采用的是ziplist结构，否则就是双向循环链表
   - Set
   - SortedSet 使用skiplist实现，用于存储用户浏览历史等场景
   - Streams（5.0新增）可以理解为只允许append的消息队列，有消费者组的概念
   - bitmap/hyperloglog/geo/等

数据结构详情：https://www.cnblogs.com/panlq/p/13098786.html

#### 5. redis如何做持久化？

redis提供了2种方式持久化：

##### 5.1 AOF
aof 全称append-only file，对于这种方式，每次对redis的写操作，都会通过后台线程记录到aof文件中，有3种回写策略来控制刷盘：

- Always 每个命令执行完后立即刷新到磁盘。**基本可以保证数据不丢失**，但是影响性能。
- Everysec 每个命令执行完后先写到缓冲区，然后定时刷盘。可能会丢数据。
- No 每个命令执行完后写到缓冲区，刷盘操作由操作系统决定。可能会丢数据。

由于AOF记录的是命令本身，所以AOF文件记录的内容会非常大，而且在故障恢复时会非常慢，因此Redis提供了AOF文件重写功能。重写时会fork出子进程合并命令，重写完成后会合并到最新的AOF文件中。

##### 5.2 RDB

rdb全称redis database，它是一种内存快照备份，对于这种方式，会fork出子进程进行快照备份。注意执行fork操作本身会阻塞线程，如果主进程的内存越大，阻塞时间越长。

在Redis4.0后提供了一个混合使用AOF和RDB的功能，具体操作是：

- 定期进行快照备份，避免频繁fork影响主进程
- aof只记录2次快照之间的操作，避免了aof文件过大而重写的开销

如果需要深入了解和对比这2种实现方式，可以参考[官方文档](https://redis.io/topics/persistence)。

> 写实复制中的写包括父进程和子进程对共享内存的写入，此时内存以页为单位进行复制，这也是为什么作者建议关闭THP的原因。
>
> 对于读多写少的场景尤为适合，但是对于读多、写多的场景呢？

#### 6. redis集群模式有哪些？

redis经历过这些模式：

- Sentinel
- Twemproxy
- Codis
- Cluster

参考：https://zhuanlan.zhihu.com/p/34592063

#### 8. 简要描述pipeline机制及优点

#### 9. 为什么Redis使用单线程设计？

参考：https://draveness.me/whys-the-design-redis-single-thread/

#### 10. redis和memcached有哪些区别？

   1. redis提供了丰富的数据结构，而memcached仅支持string类型操作。
   2. 批量操作、事务、持久化等在memcached中没有。
   3. redis是单线程模式，memcache是多线程。

#### 11. 简要描述一下缓存的更新策略有哪些？

##### 11.1 删除过期键策略

- 定时删除，内存友好，但占用CPU
- 惰性删除，CPU友好，但占用内存
- 定期删除，二者的折中

##### 11.2 缓存淘汰算法

###### 11.2.1 LRU

LRU(Least Recently Used) 表示最近最少使用，它假定的是如果一个key最近很少使用，那么将来少会被访问。

Redis使用的是近似LRU算法：随机采集法淘汰key，（如）随机采集16个key，从中淘汰最少使用的key。Redis会为每个key新增3个字节的额外空间来存储最近一次的访问时间，最少使用就是根据这个时间进行判断的。


LRU算法缺点就是它只会判断最近的访问时间：对于很少访问的key，如果最近被访问过一次，那么将不会被淘汰。反之，对于经常被访问到的key，最近很少访问，那么将会被淘汰。

###### 11.2.2 LFU

LFU(Least Frequently Used) 表示最近频繁使用，它以最近访问时间，以及访问频次最为淘汰依据，较少被访问的key优先被淘汰，反之优先保留。

##### 11.3 redis中的8种缓存淘汰策略

1. **noeviction**（默认）：不淘汰任何数据，当内存不足时，新增操作会报错
2. **allkeys-lru**：淘汰整个键值中最久未使用的键值
3. **allkeys-random**：随机淘汰任意键值
4. **volatile-lru**：淘汰所有设置了过期时间的键值中最久未使用的键值
5. **volatile-random**：随机淘汰设置了过期时间的任意键值
6. **volatile-ttl**：优先淘汰更早过期的键值
7. **volatile-lfu**：淘汰所有设置了过期时间的键值中，最少使用的键值
8. **allkeys-lfu**：淘汰整个键值中最少使用的键值

其中 allkeys-xxx 表示从所有的键值中淘汰数据，而 volatile-xxx 表示从设置了过期键的键值中淘汰数据。

> 面试过程中如果记不住，可以说个大概：不淘汰、按ttl淘汰、按lru淘汰、按lfu淘汰、随机淘汰。

#### 12. redis有哪些优化技巧？

redis监控指标：吞吐量、延迟、内存、连接数、缓存命中率。

##### 12.1 运维层面

1. TCP长连接。长连接避免了每次请求redis都创建新的连接，而创建连接的过程非常耗时
3. 网络最大连接数
4. Overcommit内存，避免Redis持久化时OOM
5. **关闭持久化**，在集群模式下RDB和AOF并不是必须的，所以可以关闭这些功能带来性能提升
6. **禁用透明大页**（Transparent Hugepages，缩写THP），透明大页本意是减少使用大页的复杂度，改进性能，但在实际使用过程中却有很多问题，如内存泄漏、卡顿、cpu使用率高等，所以多数情况（特别是针对数据库产品）需要禁用THP。

> 透明大页是有很大问题，但是具体有啥问题我记不住了，面试过程中不要提这个，否则有被K.O.的风险。

##### 12.2 服务端

1. **使用lazy free机制**。可以理解成惰性删除，即将删除操作放到单独的子线程处理，减少主线程的阻塞，可以有效的避免删除bigkey带来的性能问题
2. 设置键值过期时间，节约内存，避免频繁触发内存淘汰机制
3. **使用分布式架构**（主从、哨兵、集群）来分担读写压力

##### 12.3 客户端层面

1. 减小键值对存储长度，据实验，键值对的长度和性能成反比，具体来说，键值内容越长，序列化占用的cpu越多，网络传输的内容就越多，redis占用的内存越大导致可能频繁触发内存淘汰机制。精简数据结构，使用序列化+压缩减小长度，比如protostuff（或kryo）+snappy
2. 禁用耗时的命令，如果keys，save等
3. 使用slowlog查询耗时命令并优化
4. **使用pipeline批量查询**，极大提升性能。
5. **使用连接池**，避免频繁创建、销毁redis连接带来的性能损失

#### 13. 缓存的三大问题

##### 13.1 缓存穿透

缓存穿透指的是缓存和数据源都没有指定的数据，从而造成每次查询都会走一遍缓存和数据源，当访问量大时会造成巨大的压力。

解决方案：

- 缓存空对象，这种方式会造成缓存空间的一定量浪费，并且对于实时性较高的场景也不太适宜
- 布隆过滤器 ，这种方式记录已有的数据，它高效，但可能存在漏报、不能删除已有记录，代码维护难度较大等问题

##### 13.2 缓存击穿

缓存击穿指的是热点key会扛住大部分流量，当这部分key过期的瞬间，大量并发请求会直接打到数据源上，从而对数据源造成巨大压力。

解决方案：

- 使用锁机制，使用（分布式或单机）锁，当缓存返回null时，仅允许第一个请求读取数据源并设置缓存，之后释放锁，其他的请求再次读取缓存并从中获取数据
- 在缓存失效之前主动更新缓存，或者让热点数据永不过期

##### 13.3 缓存雪崩

缓存雪崩指的是大量的key集中失效，如redis宕机，flushdb，大量的key设置的过期时间接近（如双11零点商品缓存过期）等都会引发雪崩的问题，此时所有的请求的压力都会落到数据源上，从而造成巨大压力。

解决方案：

- 搭建高可用集群避免redis宕机
- 禁用flushdb等危险命令
- 设置不同的过期时间，防止同时失效

对于缓存雪崩的问题需要具体情况具体讨论。

#### 14. redis热key定位

如何监控热key：https://www.infoq.cn/article/3L3zAQ4H8xpNoM2glSyi

#### 15. redis代理

在看了一些大厂的文章后发现，他们的玩法都是在redis前架设redis proxy，客户端通过proxy与redis交互。这种方式性能应该会下降，但是好处似乎也有不少：

- 客户端无需关注redis服务端架构，所有请求一股脑转发给proxy，这样无论是redis拓扑改变，还是扩容等，客户端都是无感知的
- 更好的扩展，如增强redis的权限验证，过滤阻塞API等
- 方便监控，正如第14点提的那样，没有代理层很难对热key进行监控（基于良好的扩展）
- 减少redis连接压力，redis连接较多时将会消耗大量CPU来处理这些连接，使用proxy后，大量的连接在proxy上，redis的压力会小很多，也就避免了因为连接过多带来的延迟

目前开源的redis代理组件有：

- [predixy](https://github.com/joyieldInc/predixy) 性能较好
- [redis-cluster-proxy](https://github.com/RedisLabs/redis-cluster-proxy) 官方出品，似乎还不太成熟，有待生产环境验证

#### 16. 渐进式rehash

redis中的hash结构存储的数据可能会非常大，所以它的扩容的过程是分多次、渐进式的将旧hash（ht[0]表示）中的数据拷贝到新hash（ht[1]表示），具体来说包括以下步骤：

1. 分配新的空间ht[1]
2. 在字典中新增一个rehash标识，表名正在rehash
3. rehash期间，对字典的增删改查都会顺带将ht[0]的数据迁移一部分数据到ht[1]，直到迁移完成

在rehash期间，对于查询等操作会同时使用新、旧hash表。比如，对于查找操作，先会在ht[0]中查找，如果没有的话再去ht[1]查找。而对于新增操作，一律在ht[1]中操作。

> 考虑一个问题：由于rehash过程可能会比较长，那么期间ht[1]满了的话会怎样？
>
> 答：直接报错哦。

> 美团面试题。

### 面试题集合

https://zhuanlan.zhihu.com/p/112944545

### 参考

1. [Redis到底快在哪里？](https://time.geekbang.org/column/article/90221)
2. [剖析Redis常用数据类型对应的数据结构](https://time.geekbang.org/column/article/79159)
3. [为什么Redis一定要用跳表来实现有序集合？](https://time.geekbang.org/column/article/42896)
4. [压缩列表](https://redisbook.readthedocs.io/en/latest/compress-datastruct/ziplist.html)
5. [基于redis的分布式锁实现](https://juejin.im/entry/5a502ac2518825732b19a595) 大部分中文分布式锁文章（包括这篇）都有问题，这个链接放到这里是为了对比6
6. [Distributed locks with Redis](https://redis.io/topics/distlock) redis官方实现的分布式锁
7. [如何设计更优的分布式锁？](https://time.geekbang.org/column/article/125983)
8. [Redis Cluster原理初探](https://mp.weixin.qq.com/s/zjwiOkRFvQDpKfeFL1-dUQ)
9. [Redis Best Practices and Performance Tuning](https://medium.com/@abhishekbhardwaj510/redis-best-practices-and-performance-tuning-c48611388bbc)
10. [Redis 性能优化的 13 条军规！收好了](https://mp.weixin.qq.com/s?__biz=MzU4Mjk0MjkxNA==&mid=2247484686&idx=1&sn=e7f6965c28980bb7143b105394b691dd)
11. [史上最全、最新的Redis面试题（2020最新版）！](https://mp.weixin.qq.com/s?__biz=MzI0MDQ4MTM5NQ==&mid=2247492969&idx=2&sn=dc7a588bc973da6b5630b4c1996eb292&chksm=e9188075de6f09638d11dd24495231aeeec1ffd48ab6c88f4f854163afb460c4e0511365669b&mpshare=1&scene=23&srcid&sharer_sharetime=1588341053851&sharer_shareid=8b6cce4aa7804cb52b9e5a9c08be2cf4%23rd)
12. [熟悉Redis吗，项目中你是如何对Redis内存进行优化的](https://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247484460&idx=1&sn=fbe1377d2e51451311aa910c92de022a&chksm=e80db25adf7a3b4c9d3b38c5c3c73e6ce97dbbcf8c8249acddc452352bf771f28a5ad82c02b1&mpshare=1&scene=23&srcid&sharer_sharetime=1590196890113&sharer_shareid=8b6cce4aa7804cb52b9e5a9c08be2cf4%23rd)