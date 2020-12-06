#### 1. Count(*)原理

在不同的MySQL引擎中，对于一个不带where条件的Count(*)有不同的实现，具体来说：

- MyISAM引擎中把表的总行数存到磁盘中了，执行Count(*)时直接读取的该字段
- InnoDB引擎需要把数据一行行的取出来，然后进行的累加

InnoDB之所以需要这么做，是因为它是支持事务的。而事务的默认隔离级别是可重复度，所以在不同的事务中，统计到的值可能是不一样的，InnoDB没办法像MyISAM那样简单的存起来。

所以为了高效的获取表的总行数，您需要自行维护该值。专栏中记录了不同的维护方式优缺点，具体参见[《14 | count(*)这么慢，我该怎么办？》](https://time.geekbang.org/column/article/72775)。

**接下来，说说Count(*)、Count(1)、Count（主键ID）和Count(字段)之间的区别和性能**。

在进行Count统计时，内部的原理大致是先由引擎层查询符合条件的数据，然后Server层进行累加，因此：

- `Count(字段)` 遍历整张表取出该字段值，如果该字段是可空类型，还需要判断是否为null，然后返回给Server，Server会累加不为null的行数
- `Count(主键ID)` 遍历整张表取出主键字段值，返回给Server，然后累加不为null的行数
- `Count(1)` 会遍历InnoDB整张表，但不取值，Server层对于返回的每一行进行累加
- `Count(*)` MySQL并不会取出所有字段，而是做了专门的优化，所以和Count(1)相同也不会取值，Server层对返回的每一行进行累加

对于上述的查询方式，性能从高到低排序：

```mysql
count(*) > count(1) > count(id) > count(字段)
```

由于count(*)做过了特殊的优化，所以它的性能是最好的，请尽量使用它。

#### 2. order by原理

对于可以利用上索引的覆盖查询，无需排序，因为已经索引本身就是有序的。

对于无法利用上索引的查询，MySQL会创建一个名为`sort_buffer`的内存，用于存放待排序的字段并排序，内存排序一般使用**快速排序**。而如果内存里放不下所有数据，就会利用磁盘文件进行外部排序，外部排序一般使用**归并算法**：将要排序的数据分成若干份，单独排完序后再组合成一个大文件。通过以下命令查看是否使用了外部排序：

```mysql

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

这个方法是通过查看 OPTIMIZER_TRACE 的结果来确认的，你可以从 number_of_tmp_files 中看到是否使用了临时文件。

正常情况下，MySQL会优先使用**全字段排序**，全字段指的是将所有要返回的字段（select语句后的字段）放到sort_buffer中，排完序（使用快排）直接返回给客户端，这样可以避免回表操作带来的磁盘访问开销。全字段排序对内存占用较高，如果sort_buffer中要放入的字段太多导致内存放不下时，MySQL会使用**rowid排序**，即只将要排序的字段和rowid放入sort_buffer，排完序后回表取出要返回的字段给客户端。

有关order by的参数配置：

```mysql
-- 设置sort_buffer大小
SET sort_buffer_size
-- 设置全字段排序行数据长度，超过这个值将使用rowid排序
SET max_length_for_sort_data = 16;
```

这部分内容参见专栏[《16 | “order by”是怎么工作的？》](https://time.geekbang.org/column/article/73479)。

#### 3. order by之临时表

上面提到MySQL会优先使用全字段排序，但情况不总是这样。如果在SQL查询过程中需要使用临时表，而且是**内存临时表**的话，MySQL会优先使用rowid排序，因为此时没有磁盘访问开销，而且占用的内存会比全字段要少。比如这个SQL语句：

```mysql
CREATE TABLE `words` ( 
  `id` int(11) NOT NULL AUTO_INCREMENT, 
  `word` varchar(64) DEFAULT NULL, 
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

select word from words order by rand() limit 3;
```

此时，`rand()`函数将会产生一个内存临时表，表中包含2个字段：一个随机数和word字段。接下来会从words表中取出所有数据填充这个临时表，然后就是创建sort_buffer并使用rowid算法进行排序了。

当然内存临时表的也不是无限大的，通过`tmp_table_size`参数来设置最大内存临时表大小。当临时表超过了这个大小后将转成**文件临时表**。

对于上述的SQL查询语句，还有一点需要说明的是它会使用**优先队列排序**（priority queue sort）。在第2节提到的2类算法：快排和归并，都需要先将所有的数据排完序，而对于这条使用了`limit 3`的SQL来说，根本用不着全排序，只需要取最小的3个数就行了。对于这种当排序后只读取少量数据时MySQL会将使用优先排序算法进行排序。

这个算法有好也有坏，参见：

- [《17 | 如何正确地显示随机消息？》](https://time.geekbang.org/column/article/73795)
- [《MySQL · 答疑解惑 · MySQL Sort 分页》](http://mysql.taobao.org/monthly/2015/06/04/)

#### 4. join语句详情

**Index Nested-Loop Join（NLJ）**：如果可以利用上右表的索引，那么对左表做扫描（？），然后根据取出来的每行去查右表（索引查找）。对于这种情况我们应该遵循**小表驱动大表**原则，在数据量庞大的表中更能体现出这个原则的优势。

**Block Nested-Loop Join（BNL）**：如果右表无法利用索引，那么左表的数据会被加载到内存join_buffer中，然后依次和右表的每一行进行对比，将满足条件的加入结果集。如果内存放不下（join_buffer_size参数控制），那么左表的数据将会分段加载到join_buffer进行连接，此时左表表的扫描次数和总的分段数有关。

> MySQL会针对表数据量优化join语句顺序。

> 大表 join 操作对 IO 有影响，但是在语句执行结束后，对 IO 的影响也就结束了。但是，对 Buffer Pool 的影响就是持续性的（与缓存淘汰机制有关），需要依靠后续的查询请求慢慢恢复内存命中率。

Multi-Range Read 优化 (MRR)：这个优化主要目的是尽量使用顺序读盘。从回表说起，回表的过程是一行行的去主键索引查整行数据，如果需要回表的数据较多，大概率会是随机访问（回表的id顺序可能被打乱了），性能相对较差。而MRR的核心就是将对id的访问转换成有序的，具体做法就是将需要回表的id放入`read_rnd_buffer`，然后对进行排序，最后按排完序的数组依次去主键索引查找对应的行。

**Batched Key Access(BKA)**：BKA是针对NLJ的优化，具体来说就是将左表的行先放到join_buffer中，然后利用MRR算法去右表中查找满足条件的行，避免随机读取。

这部分内容参见：

- [《34 | 到底可不可以使用join？》](https://time.geekbang.org/column/article/79700)
- [《35 | join语句怎么优化？》](https://time.geekbang.org/column/article/80147)

#### 5. group by语句详情

参见：[《37 | 什么时候会使用内部临时表？》](https://time.geekbang.org/column/article/80477)

#### 6. TokuDB

TokuDB是一个开源的MySQL引擎，它适用于写多读少的场景，在大数据领域有一些应用，知乎就在对已读服务（曝光过滤）技术选型的时候使用过这个引擎。这个引擎的优缺点参见：http://mysql.taobao.org/monthly/2017/07/04/