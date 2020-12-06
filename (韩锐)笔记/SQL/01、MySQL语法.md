## 语法

此处仅列出我当前还不清楚/不熟悉的语法

- `<=>` 安全等于，可以用来判断是否相等，也能用于`NULL`值（`null`值判断只能用`IS NULL`，不能用`=`号）
- `CONCAT` 连接字符串，不能用+号
- `BETWEEN...AND...` 包含左右临界值

### 常见函数

- `length` 字节数统计
- `concat` 连接字符串
- `instr` 查找子串第一次出现的索引
- `trim` 
- `lpad`/`rpad` 左/右边填充
- `if(exp, true_val, false_val)` 判断表达式
- `case ... when ... then ...else ... end`

### 连接

- `NATURAL JOIN` 自然连接，按（相同的）列名去匹配值相等的行。自然连接还能和`LEFT/RIGHT JOIN`组合使用
- `CROSS JOIN` 交叉连接，也就是笛卡尔乘积
- `FULL OUT JOIN` 全外连接，包含左、右表所有的数据，相同的连接，不同的填充null

### 查询

- `any` 和子查询返回的某一个值比较
- `all` 和子查询返回的所有值比较

### 查询分析器

MySQL Query Optimizer。使用`explain`语法分析SQL语句查询计划，例如：

```sql
explain select * from news where category_id=687 limit 1;
```

返回的结果中包含如下字段：
- `id` id值越大表示执行优先级越高，越先执行；id值相同，从上到下顺序执行
- select_type 每个select子句的类型
- table 相关数据是关于哪个表的
- type 访问类型，性能有限顺序：system>const>eq_ref>ref>range（索引范围查找）>index（索引扫描），ALL（Full Table Scan，表扫描），index（Full Index Scan，索引扫描）
- possible_keys 可能应用到相应表的索引，
- key 实际使用到的索引（index）
- key_len 表示索引中使用的字节数，
- ref 哪些列或常量用于查找索引列的值
- rows 预估查询数据的行数
- Extra 包含了额外但非常重要的信息
  - Using filesort 说明MySQL无法利用索引完成排序，需要使用文件内排序来得到经排序的数据
  - Using temporary 使用临时表来存储中间结果集

### 索引优化

- 存储引擎不能使用范围查找条件右边的列，如`select * from news where deleted=0 and status>1 and date='20200427'`，如果为where条件各列创建索引，那么由于使用了范围查找，所以索引的date列没有被查询引擎使用
- 尽量使用覆盖所有，避免使用`select *`
- 在使用不等于（`!=`或`<>`）条件时将无法使用索引
- 对`col1 like %xxx%`列建立覆盖索引，也能利用上索引
- 少用or语句，用它来连接时索引会失效
- 小表（数据量小）驱动大表（数据量大），连接时小表放左边，大表放右边，并且此时`exists`性能优于`in`
- 如果where最左前缀定义为常量，则order by可以使用索引，比如`where a=const order by b,c`（对a,b,c建索引）

### 事务

```sql
--开启事务，针对当前连接有效
set autocommit=0;

start transaction;  --可选
...
commit;  --提交
rollback; -- 回滚

```



## 原理

### 存储引擎

| 对比项 | MyISAM                     | InnoDB                                               |
| ------ | -------------------------- | ---------------------------------------------------- |
| 主外键 | /                          | 支持                                                 |
| 事务   | /                          | 支持                                                 |
| 行级锁 | 表级锁，不支持高并发       | 行级锁，适合高并发                                   |
| 缓存   | 只缓存索引，不缓存真实数据 | 缓存索引、真实数据，对内存要求较高，内存大小决定性能 |
| 表空间 | 小                         | 大                                                   |
| 关注点 | 性能                       | 事务                                                 |

### 事务

### 性能优化

1. 索引优化步骤
   1. 优化sql语句，减小冗余
   2. 检查索引是否失效
   3. 关联查询太多join，设计缺陷
   4. 服务器调优及各参数优化（缓冲、线程数量）

## 问题

1. `COUNT(*)`和`COUNT(1)`哪个效率高？为什么？
2. 字符集编码
3. 什么是索引？



## 参考

1. https://www.bilibili.com/video/BV12b411K7Zu
2. https://www.cnblogs.com/xuanzhi201111/p/4175635.html
3. https://dev.mysql.com/doc/refman/8.0/en/explain-output.html