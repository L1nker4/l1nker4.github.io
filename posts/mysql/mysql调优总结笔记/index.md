# MySQL调优总结




# 前言



数据库调优的几个维度：

- 建立索引
- SQL语句优化
- 服务器参数调优：包括缓冲区、线程数等
- 分库分表、集群模式



调优目标：

- 尽可能节省系统资源，以提高系统吞吐量。
- 合理的结构设计和参数调整，以提高用户操作响应的速度。
- 减少系统的瓶颈，提高MySQL整体性能。



如何定位调优问题：

- 用户反馈
- 日志分析
- 服务器资源监控



调优维度：

1. 选择合适的DBMS
2. 优化表设计
   1. 遵循三范式的原则
   2. 多表联查可以考虑反范式化
   3. 表字段的数据类型选择
3. 优化SQL查询（逻辑）
4. 使用索引（物理）
5. 使用缓存
6. 库级优化
   1. 读写分离
      1. 一主一从
      2. 双主双从
   2. 数据分片：对数据库进行分库分表。



# MySQL服务器优化



两个方面：

1. 硬件优化
2. MySQL服务的参数优化



## 硬件调优

- 配置较大的内存，增加缓冲区容量，减少磁盘IO。
- 配置高速磁盘系统，减少磁盘IO的时间。
- 合理分布磁盘IO，将磁盘IO分布在多个设备上，减少竞争。
- 配置多处理器。



## 参数调优



通过优化MySQL可以提高资源利用率，从而提高MySQL服务器性能。

几个重要的参数：

- innodb_buffer_pool_size：表和索引的缓存区大小。
- key_buffer_size：索引缓冲区大小，所有线程共享，值太大也会导致OS频繁换页。
- table_cache：同时打开的表的个数。
- query_cache_size：**查询缓冲区**大小，与query_cache_type配合使用。
- query_cache_type：0代表所有查询不使用查询缓冲区，1表示所有都使用，当查询语句指定`SQL_NO_CACHE`则不使用。
- sort_buffer_size：每个需要进行**排序**的线程分配的缓冲区大小，增加这个参数的值可以提高`ORDER BY`或`GROUP BY`操作的速度。
- join_buffer_size：每个需要**联合查询**的线程所使用的缓冲区大小。
- read_buffer_size：每个线程**连续扫描时**为扫描的每个表分配的缓冲区的大小。
- innodb_flush_log_at_trx_commit：**何时将redo log buffer的数据写入redo log file，并将日志文件写入磁盘中。**默认为1。
  - 0：redo log buffer每隔一秒将其数据刷入page cache，该模式下事务提交不会触发刷盘操作。
  - 1：每次事务提交都会将将redo log buffer中数据刷入page cache，并立刻刷入磁盘。效率较低也为安全。
  - 2：每次事务提交都会将redo log buffer中数据刷入page cache，由OS同步到磁盘。（每秒一次）
    - mysql进程崩溃不会有数据丢失，当时OS宕机会有数据丢失。
- innodb_log_buffer_size：InnoDB存储引擎的**事务日志缓冲区**，为了提升性能，也是先将信息写入 `Innodb Log Buffer` 中，当满足 `innodb_flush_log_trx_commit` 参数所设置的相应条件（或者日志缓冲区写满）之后，才会将日志写到文件（或者同步到磁盘）中。
- max_connections：允许连接到MySQL数据库的最大数量。如果`connection_errors_max_connections`不为0，并且一直增长，说明不断有连接因为数据库连接数已到最大值而失败，此时考虑增大`max_connections`的值。
- back_log：用于**控制MySQL监听TCP端口时设置的积压请求栈大小。** 连接数达到`max_connections`，新来的请求将会被存在堆栈中，以等待某一连接释放资源，如果等待连接的数量超过back_log，将会报错。
- thread_cache_size：线程池缓存线程数量的大小，当客户端断开连接后将当前线程缓存起来， 当在接到新的连接请求时快速响应无需创建新的线程 。这对于短链接的应用程序十分有用。
- wait_timeout：一个连接的最大连接时间。
- interactive_timeout：服务器在关闭连接前等待行动的秒数。



# 优化数据库结构

几个策略：

1. 拆分表：冷热数据分离
2. 增加中间表
3. 增加冗余字段
4. 优化数据类型
   1. 整数类型优化：使用INT类型。
   2. 避免使用TEXT、BLOB数据类型
   3. 避免使用ENUM类型：Order By效率较低
   4. 使用TIMESTAMP存储时间：存储空间小，4字节
   5. 用DECIMAL代替FLOAT和DOUBLE存储精确浮点数
5. 优化插入记录的速度
6. 使用非空约束
7. 分析表：更新关键字的分布
   1. 分析表过程中，会对表加上一个只读锁。
   2. 统计结果会反映在**cardinality**的值上，该值统计了某一个键所在列不重复值得个数，cardinality可以通过 `SHOW INDEX FROM 表名`查看。
8. 检查表：检查表中是否存在错误。
   1. 检查表过程中，会对表加上一个只读锁。
9. 优化表：对于变长字段（VARCHAR、BLOB、TEXT等列）进行了更新，优化表可以来整理数据文件得碎片。







# 索引优化与查询优化





## 查询性能分析



### 查看系统性能参数



在MySQL中，可以使用 `SHOW STATUS` 语句查询一些MySQL数据库服务器的性能参数 、 执行频率 。

一些常用的性能参数如下： 

- Connections：连接MySQL服务器的次数。
- Uptime：MySQL服务器的上 线时间。
- Slow_queries：慢查询的次数。
- Innodb_rows_read：Select查询返回的行数 
- Innodb_rows_inserted：执行INSERT操作插入的行数
- Innodb_rows_updated：执行UPDATE操作更新的 行数
-  Innodb_rows_deleted：执行DELETE操作删除的行数
- Com_select：查询操作的次数。
- Com_insert：插入操作的次数。对于批量插入的 INSERT 操作，只累加一次。
- Com_update：更新操作 的次数。
-  Com_delete：删除操作的次数。



### 统计SQL的查询成本

查上一次查询的代价，是IO和CPU开销的总和，可以作为评价一次查询的执行效率的常用指标。

```sql
SHOW STATUS LIKE &#39;last_query_cost&#39;;
```



### 定位慢SQL



开启slow_query_log

```sql
set global slow_query_log=&#39;ON&#39;;
```



修改long_query_time阈值

```sql
show variables like &#39;%long_query_time%&#39;;

set global long_query_time = 1;
```



查询当前系统中有多少条慢查询记录

```sql
SHOW GLOBAL STATUS LIKE &#39;%Slow_queries%&#39;;
```



### EXPLAIN

EXPLAIN返回SQL语句的执行计划。

```sql
EXPLAIN SELECT select_options
or
DESCRIBE SELECT select_options
```



查询结果列如下：

- id：决定每个表的加载和读取顺序。
  - id相同，可以认为是一组，从上往下顺序执行
  - id越大，优先级越高，越先执行
  - 不同id代表单独一趟查询，趟数越少越好
- table：一趟查询在哪个表中查询
- select_type：查询类型
  - SIMPLE：简单select查询，不包含子查询或者UNION
  - PRIMARY：嵌套查询最外层的部分
  - SUBQUERY：出现在select或者where后面中的子查询被标记为SUBQUERY
  - DERIVED：在from后面的子查询
  - UNION：将UNION后面的select标记位UNION
  - UNION RESULT：从 UNION表获取结果的select
- type：数据的访问方法，结果值从好到坏依次为：system &gt; const &gt; eq_ref &gt; ref &gt; fulltext &gt; ref_or_null &gt; index_merge &gt; unique_subquery &gt; index_subquery &gt; range &gt; index &gt; ALL，SQL 性能优化的目标：至少要达到 range 级别，要求是 ref 级别，最好是 const级别。（阿里巴巴 开发手册要求）
  - system：表中只有一行记录
  - const：通过索引一次就找到了
  - eq_ref：唯一性索引扫描
  - ref：使用普通索引或者唯一索引的部分前缀
  - range：只检索给定范围的行，使用一个索引来选择行。
  - index：full index scan，index类型只遍历索引树
  - all：全表扫描
- possible_keys：可能使用的索引
- key：实际使用的索引
- key_len：索引中使用的字节数
- ref：当使用索引列等值查询时，与索引列进行等值匹配的对象信息，取ref（字段）或const（常量）
- rows：查询大约所需读取的行数
- Extra：额外信息
  - Using filesort：无法使用索引完成，只能在内存或磁盘中进行排序。
  - Using temporary：MySQL需要创建一张临时表来处理查询，常见于order by 和 group by。
  - Using index：使用了覆盖索引，效率高，如果同时出现`Using where`
  - Using index condition：使用了索引下推





### optimizer trace

```sql
set optimizer_trace=&#34;enabled=on&#34;,end_markers_in_json=on;
set optimizer_trace_max_mem_size=1000000;


select * from student where id &lt; 10;

select * from information_schema.optimizer_trace\G
```



### sys.schema

```sql
#1. 查询冗余索引
select * from sys.schema_redundant_indexes;
#2. 查询未使用过的索引
select * from sys.schema_unused_indexes;
#3. 查询索引的使用情况
select index_name,rows_selected,rows_inserted,rows_updated,rows_deleted
from sys.schema_index_statistics where table_schema=&#39;dbname&#39; ;


# 1. 查询表的访问量
select table_schema,table_name,sum(io_read_requests&#43;io_write_requests) as io from
sys.schema_table_statistics group by table_schema,table_name order by io desc;
# 2. 查询占用bufferpool较多的表
select object_schema,object_name,allocated,data
from sys.innodb_buffer_stats_by_table order by allocated limit 10;
# 3. 查看表的全表扫描情况
select * from sys.statements_with_full_table_scans where db=&#39;dbname&#39;;


#1. 监控SQL执行的频率
select db,exec_count,query from sys.statement_analysis
order by exec_count desc;
#2. 监控使用了排序的SQL
select db,exec_count,first_seen,last_seen,query
from sys.statements_with_sorting limit 1;
#3. 监控使用了临时表或者磁盘临时表的SQL
select db,exec_count,tmp_tables,tmp_disk_tables,query
from sys.statement_analysis where tmp_tables&gt;0 or tmp_disk_tables &gt;0
order by (tmp_tables&#43;tmp_disk_tables) desc;


#1. 查看消耗磁盘IO的文件
select file,avg_read,avg_write,avg_read&#43;avg_write as avg_io
from sys.io_global_by_file_by_bytes order by avg_read limit 10;


#1. 行锁阻塞情况
select * from sys.innodb_lock_waits;
```





## SQL优化



SQL优化大方向主要分为以下两块：

- 物理查询优化：通过**索引**和**表连接**方式等技术来优化。
- 逻辑查询优化：通过**SQL等价变换**提升查询效率。





### JOIN语句原理

join本质就是**各个表之间数据的循环匹配**，MySQL5.5之前，使用嵌套循环方式（Nested Loop Join），在MySQL5.5之后，引入BNLJ算法来优化嵌套执行。

驱动表为主表，被驱动表为从表，小表驱动大表。通过`explain`关键字查看。



#### Simple Nested-Loop Join

从驱动表A取出数据，遍历扫描表B，将匹配到的数据放到result中，以此类推，驱动表A，直到所有记录被遍历结束。性能消耗很大。





#### Index Nested-Loop Join

优化思路主要是为了**减少内层表数据的匹配次数**，所以要求被驱动表上必须有索引才行。通过外层表匹配条件直接与内层表索引进行匹配，减少了和内层表中每条记录去进行比较，这样大大提高了查询效率。





#### Block Nested-Loop Join

BNLJ引入了`Join Buffer`缓冲区，将驱动表数据按块缓存在`buffer`中然后全表扫描被驱动表，被驱动表的每一条记录一次性和`buffer`中所有记录进行匹配，降低了被驱动表的访问频率。

```sql
# 查看block_nested_loop状态，默认开启
show VARIABLES like &#34;%optimizer_switch%&#34;;

# 查看join buffer的大小，默认为256KB
show VARIABLES like &#34;%join_buffer%&#34;;
```



#### hash join



- MySQL8.0.20开始废弃BNLJ，从MySQL8.0.18开始引入了Hash Join。

- Hash Join是做**大数据集连接时**的常用方式，优化器将较小的表利用`Join Key`在内存中建立哈希表，然后扫描大表并且探测哈希表，找到和哈希表匹配的行。
  - 适用于小表可以完全放在内存中，总成本：访问两个表的成本之和。
  - 表很大无法放入内存时，优化器将它分割成若干不同的分区，不能放入内存的部分写到磁盘的临时段。
  - 能很好的工作于没有索引的大表和并行查询的环境，但是只适用于等值连接



#### 总结

- 整体效率：INLJ &gt; BNLJ &gt; SNLJ
- 用小结果集驱动大结果集，**本质**是：**减少外层循环的数据层数**。

- 被驱动表的JOIN字段已经创建了索引。
- JOIN字段的数据类型保证绝对一致。
- LEFT JOIN 时选择小表为驱动表，INNER JOIN 时MySQL自动选择小表为驱动表。





### 子查询优化

子查询可以一次完成逻辑上需要多个步骤才能完成的SQL操作，但是子查询的执行效率不高，原因如下：

- 执行子查询时，MySQL需要为内层语句的结果建立一张临时表，然后外层语句从该表继续查询，完毕后再撤销临时表，这样会消耗很多系统资源。
- 子查询产生的临时表不存在索引，故查询性能受限。

使用JOIN查询代替子查询，并且可以考虑使用索引优化JOIN查询。

&gt; 尽量不要使用NOT IN 或者 NOT EXISTS，用LEFT JOIN xxx ON xx WHERE xx IS NULL替代



### 排序优化

在MySQL中，支持`FileSort`和`Index`两种排序方式。

- `Index`排序中，索引可以保证有序性，不需要再进行排序。
- `FileSort`一般在内存中进行排序，效率较低。



优化策略：

1. 在ORDER BY子句中使用索引，避免`FileSort`排序。
2. 无法使用`Index`时，需要对`FileSort`进行调优。
3. where和order by使用最左前缀原则，且order by的多列同DESC或者ASC顺序



filesort的两种算法：

- 双路排序：读取行指针和order by列，对它们进行排序，然后扫描已经排序好的数据，再次读取其他所需字段。需要两次磁盘IO。
- 单路排序：从磁盘读取查询需要的所有列，按照order by列进行排序，然后再输出。
  - 单路排序存在的问题：`sort_buffer`需要占用较多空间。
  - 优化策略：
    - 1. 尝试提高`sort_buffer_size`
      2. 尝试提高`max_length_for_sort_data`
      3. order by时写清楚所需要的字段，禁止`*`





### group by优化



优化原则：

- group by使用索引的原则和order by几乎一致。
- group by先排序再分组，遵循最左前缀原则。
- where效率高于having，尽量使用where进行数据过滤
- 包含了group by的查询语句，where过滤后的结果集尽量保持在1000行左右。







### 优化分页查询



当`limit 2000000,10`时，MySQL获取前2000010条数据，但是只返回后十条记录，其他全部丢弃，查询的代价特别大。



优化思路主要有以下几种方案：



在索引上完成分页操作，最后根据主键回表查询其他列内容。

```sql
EXPLAIN SELECT * FROM student t,(SELECT id FROM student ORDER BY id LIMIT 2000000,10) a WHERE t.id = a.id;
```



将`limit`查询转换到指定主键的查询（需要主键自增的要求）

```sql
EXPLAIN SELECT * FROM student WHERE id &gt; 2000000 LIMIT 10;
```





### 优先考虑覆盖索引

覆盖索引：一个索引包含了查询所需要的数据，避免了回表操作。

- 优点
  - 避免了InnoDB的回表操作
  - 可以将随机IO变成顺序IO加快查询效率
- 缺点：
  - 建立覆盖型的索引可能造成冗余。



### 字符串添加索引

MySQL支持前缀索引，不指定前缀长度时，索引默认包含整个字符串。

```sql
mysql&gt; alter table teacher add index index1(email);

mysql&gt; alter table teacher add index index2(email(6));
```

前缀索引和整个字符串索引的区别如下：

1. 使用前缀索引更加节省空间，定义长度时考虑区分度。
2. 前缀索引不支持覆盖索引，需要综合考虑。



### 索引下推(ICP)



**Index Condition Pushdown**是MySQL 5.6中引入的新特性。

未使用ICP的工作过程：

1. storage层将满足index key条件的索引记录对应的整行取出，返回给server层。
2. 对返回的数据，使用where条件进行过滤。



ICP工作过程：

1. 将满足index key条件满足的索引记录取出，使用`index filter`对where条件进行过滤，将满足条件的记录返回给server层。
2. server层对返回的数据使用`table filter`条件进行过滤。



成本差别在于：**ICP在存储引擎层通过ICP筛选掉不符合条件的数据，减少了回表成本。**



使用条件：

- 只能用于二级索引
- ICP仅适用于索引列（通常是联合索引）
- ICP可以用于MyISAM和InnnoDB存储引擎





### 其他优化策略



- EXISTS和IN的区分：小表驱动大表原则，A表小用EXISTS，B表小用IN。
- COUNT(*)、COUNT(column)、COUNT(1)的效率对比
  - COUNT(*)、COUNT(1)：两者没有本质区别，都是对所有结果进行COUNT。
    - MyISAM只需要$O(1)$复杂度，因为每张数据表都有`row_count`存储行数，一致性由表级锁保证。
    - InnoDB存储引擎采用行级锁&#43;MVCC机制，故需要**扫描全表**，需要$O(n)$复杂度。
  - COUNT(column)：尽量采用二级索引，二级索引的空间比聚簇索引小。
- SELECT(*)：禁止使用
  - 无法使用覆盖索引
  - 增加查询分析器解析成本：通过查询数据字典将`*`转换为所有列名。
  - 无用字段增加消耗，例如BLOB等类型会有行溢出额外查询消耗。
- LIMIT 1：针对全表扫描的情况，确定结果集只有一条，可以使用`LIMIT 1`优化，如果字段已经存在唯一索引，可以通过索引进行查询，则不需要`LIMIT 1`进行优化。
- 使用COMMIT释放资源
  - 回滚段上用于恢复数据的信息
  - 被程序语句获得的锁
  - redo、undo中的buffer空间







## 数据库设计



### 主键的设计



自增ID存在的问题：

- 可靠性不高：存在自增ID回溯问题，MySQL8.0修复
- 安全性不高：`/user/{id}`这样的接口会导致恶意请求爬取相应数据。
- 性能消耗：在服务器端生成
- 交互多：业务还需要额外执行一次类似`last_insert_id()`的行数才能知道刚才插入的自增值，需要额外的网络交互。
- 局部唯一性：自增ID只能在当前数据库实例中唯一，不能保证分布式系统中的唯一性。



### UUID

UUID占用36字节，MySQL中UUID由以下几部分组成：

```
UUID = 时间&#43;UUID版本（16字节）- 时钟序列（4字节）- MAC地址（12字节）
另外加上4个短横线，共36字节
```

MySQL8.0将时间低位和高位互换位置，构成了有序UUID。同时将UUID改为二进制存储，只需16字节存储。

可以通过`uuid_to_bin(@uuid, true)`将原来的UUID转换为有序递增UUID。



### 开源组件

例如美团开发的Leaf分布式ID生成等。



### 反范式化

- 为了满足性能目标，性能比规范化更重要。
- 在表中增加冗余字段，以大量减少从中搜索的时间。
- 存储空间增加，一个表的字段修改，另一个表的冗余字段也需要同步修改，否则数据不一致。
- 只有大幅度提高查询效率，才考虑反范式化。
- 常用于数仓设计，数仓通常用于存储历史数据，增删改的要求不高。





# 大表优化



## 限制查询范围

**禁止使用不带任何限制数据范围条件的查询语句。**



## 读写分离

- 一主一从模式
- 双主双从模式



## 分表



### 垂直拆分

当数据量级达到**千万级**以上时，可以键数据库切分成多块，放到多台数据库服务器中。

- 垂直分库：将关联的数据表放在同一台数据库中。
- 垂直分表：将一张数据表拆分成多张数据表，按列划分。



- **优点**：可以使得列数据变小，在查询时减少读取的Block数量，减少IO次数。
- **缺点**：主键会冗余，并且会引起JOIN操作，增加业务复杂度。



### 水平拆分



- 尽量控制单表数据量大小，保持在**1000万条**以内。
- 水平分表仅是解决**单一表数据量过大**的问题。
- 尽量不要对数据分片，拆分会带来逻辑、部署、运维的复杂度。



### 代理方案

- 客户端代理：分片逻辑在应用层，封装在JAR中，通过修改或封装JDBC层来实现，例如Sharding-JDBC等。

- 中间件代理：在应用层和数据层中间加了个代理层，分片逻辑统一维护在中间件中，例如Mycat等。



---

> Author:   
> URL: http://localhost:1313/posts/mysql/mysql%E8%B0%83%E4%BC%98%E6%80%BB%E7%BB%93%E7%AC%94%E8%AE%B0/  

