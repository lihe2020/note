### 1. 一条SQL查询语句的执行流程

<img src="../images/mysql-query-flow.png" alt="img" style="zoom:30%;" />

- 客户端发起请求
- 连接器负责接收和维护请求，相关内存只在连接断开后释放，所以为避免OOM，需要定期或主动断开连接
- 查询缓存缓存了相同SQL语句的上一次执行结果，命中则直接返回。但大都时候命中率都比较低，所以建议禁用
- 分析器用于词法分析，语法分析，对于语法的正确性会在此阶段验证
- 优化器用于根据查询语句选择索引、优化SQL的执行顺序等
- 执行器负责执行SQL语句，它会通过接口调用存储引擎（InnoDB）去执行相关逻辑

#### 1.1 InnoDB之buffer pool的缓存淘汰策略

buffer pool是数据页在内存的映射，buffer pool（BP）的写入详情见本文2.2小节。这里记录的是当buffer pool内存不够用时内存页是如何被淘汰的。

当对大表进行全表扫描时就需要将所有的数据页加载到内存，如果内存占用满了，就需要淘汰掉旧的数据页，InnoDB是使用改进版的LRU算法来实现这个功能的。

具体来说，它会将它会将所有的内存页放到一个链表结构中，链表头部的5/8称为young区，尾部的3/8称为old区。当有新的页需要加入时，首先会被插入到old区首位，如果这个页在链表中存活超过1秒，那么将会被移动到链表头部，否则不动。这样的区分是为了避免诸如全表扫描等场景对正常页产生影响。

对于全表扫描的场景，需要加载很多页到内存，这些页只会短暂使用，这时候它们只会在old区停留并很快被清除掉。更重要的是他们并不会影响young区的内存页。

当然事无绝对，如果是全表扫描的数据量过大，可能会影响正常业务的数据页进入young区，young区的数据也没法被合理的淘汰。

P.S. buffer pool有一个重要的指标：**内存命中率**。可以在`show engine innodb status`命令中查看到相关统计。

这部分内容参考：[《33 | 我查这么多数据，会不会把数据库内存打爆？》](https://time.geekbang.org/column/article/79407)。

### 2. 一条SQL更新语句是如何执行的

#### 2.1 通用概念介绍

在描述如何执行之前，需要先了解一些通用的概念：

##### 2.1.1 WAL(Write-Ahead Log)

日志先行（Write-Ahead Log）技术广泛的应用于现代数据库，指的是执行事务时，先修改对应的内存页（此时该内存页称为脏页），再把该修改行为记录到事务日志中，事务是顺序IO所以性能较高，待事务日志持久化后，脏页会被刷新到磁盘。这种技术保证了数据不丢失的情况下，进一步提升了数据库的性能。用户对数据进行了修改，必须要保证日志先于数据落盘，且只要日志落盘后，就可以给用户返回操作成功。

> WAL是一种通用的技术统称，不用的数据库可能会有不同的实现。

WAL有2个核心的步骤：

- 由于所有对数据的修改都需要写日志，当并发量很大时必然导致日志写入非常频繁；为了避免性能问题，通常会先将日志写入到缓冲区，然后再刷盘。
- 当日志写入成功后将直接返回客户端操作成功，而此时用户数据可能还没有落盘，这就需要数据库有一套刷数据的机制，称为刷脏页算法。

上述步骤在不同的MySQL版本都有不同的优化和改进，具体参考阿里文章：[《MySQL · 引擎特性 · WAL那些事儿》](http://mysql.taobao.org/monthly/2018/07/01/)。

日志先行中的日志在InnoDB里面对应的就是redo log（重做日志）。

##### 2.1.2 两阶段提交（2PC）

2PC同常见于分布式系统中，具有强一致性，是CP系统的典型实现。

在InnoDB中，因为同时使用了redo log和binlog，为了确保这2种日志写入的一致性，就需要使用2PC了。

```mysql
update T set c=c+1 where ID=2;
```

以上面的语句为例，在执行过程中分成以下步骤：

<img src="../images/mysql-2pc.jpg" alt="mysql-2pc" style="zoom:33%;" />

> 深色部分在执行器中执行，浅色部分在InnoDB中执行。

> 上述过程中的commit和事务的commit不一样，这里的commit指的是整个事务的一个步骤，并且是最后一个步骤，当执行完成后，整个事务就提交完成了

2PC指的就是上述步骤中写入redo log和binlog部分。

##### 2.1.3 写文件步骤

写文件包括write（写磁盘）、持久化（fsync）。write表示内容写到了文件系统的page cache中，但还没有写入磁盘，所以速度较快；fsync表示真正的持久化，成为hard disk，这个操作是非**常耗时**的。

#### 2.2 一条更新语句执行的详情

这里简单罗列，然后依次讲述：

1. 将数据更新到InnoDB的buffer pool
2. 将数据更新到redo log pool
3. 将数据写入到binlog
4. 所有操作成功，将redo log pool中的数据进行持久化，提交事务

##### 2.2.1 buffer pool写入详情

在写日志前会先将数据写入InnoDB的**[buffer pool](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool.html)**（BP），而在写buffer pool时，按将要修改的数据在内存页中是否存在分成2种情况：
- 存在，那么直接修改相应的内存页，此时内存页和磁盘页数据不一致（称为**脏页**），脏页将来的某一时刻会被刷到表空间（table space）
- 不存在时，按照插入的页是否是唯一索引分成2种情况：
  - 唯一索引，那么从磁盘读取相应数据页，并进行修改
  - 非唯一索引（也称，二级索引、普通索引等），那么将要修改的记录到**[change buffer](https://dev.mysql.com/doc/refman/5.7/en/innodb-change-buffer.html)**

change buffer是buffer pool内的一块内存区域，当要修改的页记录不在buffer pool中时，InnoDB就会将这些修改记录到change buffer，这样就避免了从磁盘中加载数据页产生的随机IO开销。

change buffer中的记录会在后台定期将记录合并到原数据页，这个合并的过程称为merge操作。访问buffer pool中的相应数据页（不存在）也会触发merge操作，此时会从磁盘加载数据页，并和change buffer进行merge。

对于唯一索引来说，由于需要确保唯一性，所以当内存页不存在时，需要从磁盘加载数据页判断是否记录已存在，所以这种情形下无法利用change buffer减少随机IO开销。这也是通常情况下**不建议使用唯一索引**的原因之一。

> 关于刷脏页的原理和策略参见专栏[《12 | 为什么我的MySQL会“抖”一下？》](https://time.geekbang.org/column/article/71806)。

##### 2.2.2 binlog写入详情

一个事物的binlog是不能被拆开的，因此无论这个事务有多大，都需要一次性写入。而为了做这个保证，系统为每个线程分配了一个内存区域称为**binlog cache**，事务提交时会将binlog cache写入磁盘并清空这部分内存。

MySQL提供了`sync_binlog`参数用来控制磁盘写入时机：

- `0` 每次提交事务只write，不fsync
- `1` 每次提交事务都fsync
- `n`（大于1） 表示每次事务提交都write，但是累积到n个事务后再执行fsync

在出现性能瓶颈时将`sync_binlog`设置成一个较大的值可以提升性能。但是也需要承担binlog丢失事务的风险。

相关SQL语句：

```mysql
# 查看当前服务器使用的biglog文件及大小
show binary logs;

# 事件查询命令
# IN 'log_name' ：指定要查询的binlog文件名(不指定就是第一个binlog文件)
# FROM pos ：指定从哪个pos起始点开始查起(不指定就是从整个文件首个pos点开始算)
# LIMIT [offset,] ：偏移量(不指定就是0)
# row_count ：查询总条数(不指定就是所有行)
show binlog events [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];

# 查看 binlog 内容
show binlog events;

# 查看具体一个binlog文件的内容 （in 后面为binlog的文件名）
show binlog events in 'master.000003';
```



##### 2.2.3 redo log写入详情

redo log写入前首先会将数据写入InnoDB的**[buffer pool](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool.html)**，此时内存页和磁盘页数据不一致（称为**脏页**），脏页将来的某一时刻会被刷到表空间（table space），而这些操作与redo log无关，在此列出只是为了说明是如何写入到db文件的。

接下来更新的内容被写入到**redo log buffer**，待执行commit语句时，将redo log buffer的内容持久化到redo log文件中（写文件的步骤在见2.1）。与binlog类似，为了控制redo log的写入策略，InnoDB提供了`innodb_flush_log_at_trx_commit`参数：

- `0`，每次事务提交只将redo log写入到redo log buffer中
- `1`，每次事务提交都将redo log持久化到磁盘
- `2`，每次事务提交只把redo log写到page cache

InnoDB有一个后台线程，每隔1秒会把redo log buffer中的日志通过write和fsync持久化到磁盘。所以redo log buffer不一定是在事务提交后才持久化的。此外，redo log buffer空间不够、并行事务提交也会导致commit之前持久化，详情参见极客专栏[《MySQL是怎么保证数据不丢的》](https://time.geekbang.org/column/article/76161)。

> MySQL实战45讲整体上五星好评，但是日志相关的讲解我觉得不到位，分散而且不够全面。特别是在讲到粉板这个例子后我更加迷惑了。查了很多资料，涉及的概念很多，我无法甄别对错，所以只能先一一记录了。

##### 2.2.4 组提交（group commit）机制

### 3. 结果集是如何发送到客户端的

MySQL对结果集是“**边读边发**”，也就是说不会等到查询完毕才一次性发送所有数据。具体来说，包含以下步骤：

- 获取到一行数据后，发送到**net_buffer**，这个内存大小由net_buffer_length控制，默认是16k
- 重复获取行，直到net_buffer写满，此时会调用网络接口发送出去
- 如果发送成功则清空net_buffer，继续往下读取；如果发送失败可能是网络接口满了，需要等待网络栈重新可写，再继续发送
- 网络接口缓冲区大小由`/proc/sys/net/core/wmem_default`配置决定

如果网络栈写满了，通过`show processlist`可以看到连接状态是`Sending to client`。

这部分内容参考：[《33 | 我查这么多数据，会不会把数据库内存打爆？》](https://time.geekbang.org/column/article/79407)。

### 4. 参考

1. [02 | 日志系统：一条SQL更新语句是如何执行的？](https://time.geekbang.org/column/article/68633)
2. [15 | 答疑文章（一）：日志和索引相关问题](https://time.geekbang.org/column/article/73161)
3. [23 | MySQL是怎么保证数据不丢的？](https://time.geekbang.org/column/article/76161)
4. 

