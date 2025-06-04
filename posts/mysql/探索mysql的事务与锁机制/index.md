# 探索MySQL的事务与锁机制




# 事务概念

简单来说，事务就是要保证一组数据库操作，要么全部成功，要么全部失败。MySQL中事务支持是在存储引擎层实现的。事务拥有四个重要的特性：原子性、一致性、隔离性、持久性，简称为ACID特性，下文将逐一解释。



## ACID特性

- 原子性（Atomicity）
  - 事务开始后所有操作步骤，要么全部完成，要么全部不做，不存在只执行一部分的情况。
- 一致性（Consistency）
  - 事务执行前后，数据从一个合法性状态变换到另一个合法性状态。
    - A、B转账业务，总金额不变。
  - 分为数据一致性和约束一致性。
- 隔离性（Isolation）
  - 在一个事务未执行完毕时，其它事务无法读取该事务的数据。
  - MySQL通过锁机制来保证事务的隔离性。
- 持久性（Durability）
  - 事务一旦提交，数据将被保存下来，即使发生宕机等故障，数据库也能将数据恢复。
  - MySQL使用`redo log`来保证事务的持久性。当通过事务对数据进行修改时，首先会将操作记录到`redo log`中，然后对数据库对应行进行修改，这样即使数据库宕机，也能通过`redo log`进行恢复。

ACID关系如下图所示：
![ACID关系](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/%E4%BA%8B%E5%8A%A1%E7%89%B9%E6%80%A7.png)



## 显式事务



开始事务：

```sql
BEGIN;  
或
START TRANSACTION;
```

两者区别：

- `START TRANSACTION`后面可以跟随几个修饰符：
  -  READ ONLY：标识为只读事务，该事务只能读取数据。
  - READ WRITE：标识为读写事务，该事务可以读写数据。
  - WITH CONSISTENT SNAPSHOT ：启动一致性读。



完成事务：

```sql
# 提交事务
COMMIT;

#回滚事务
ROLLBACK;

#将事务回滚到某个保存点。
ROLLBACK TO [SAVEPOINT]
```



## 隐式事务

```sql
SHOW VARIABLES LIKE &#39;autocommit&#39;;
```



隐式提交数据的情况：

1. 数据定义语言：CREATE、ALTER、DROP
2. 隐式修改mysql数据库中的表
3. 事务控制（连续两次BEGIN，第一个BEGIN后面的语句会自动提交）或关于锁定的语句
4. 加载数据的语句
5. MySQL复制的语句





## completion_type



```sql
set @@completion_type = 1;
```

该变量有三种取值：

- 0：默认值，当我们执行COMMIT时会提交事务，再执行下一个事务时，还需要使用BEGIN来开启。
- 1：提交事务后，相当于执行了`COMMIT AND CHAIN`，开启链式事务，当我们提交事务后会开启一个相同隔离级别的事务。
- 2：相当于`COMMIT AND RELEASE`，提交事务后，与服务器断开连接。



## 事务分类

- 扁平事务：最简单的一种，使用BEGIN开启，由COMMIT或ROLLBACK结束。
- 带有保存点的扁平事务：支持回滚到指定保存点的事务。
- 链式事务：一个事务由多个子事务构成，提交前一个事务，触发下一个事务。
- 嵌套事务：由顶层事务控制下面各个层次的事务。
- 分布式事务：分布式系统中的扁平事务。



# 并发事务问题





## 脏写

事务A覆盖了事务B未提交的更新数据。

任何隔离级别都可以解决该问题。



## 丢失更新

**丢失更新**就是两个事务在并发下同时进行更新，后一个事务的更新覆盖了前一个事务更新的情况。这是一种并发写入的问题。基本情景如下表：

| 时间 |  事务A   |  事务B   |
| :--: | :------: | :------: |
|  1   | 开启事务 | 开启事务 |
|  2   | a = 100  | a = 100  |
|  3   |    /     | a -= 10  |
|  4   |    /     |  commit  |
|  5   |    /     |    /     |
|  6   | a &#43;= 20  |    /     |
|  7   |  commit  |    /     |



## 脏读

一个事务读取了另一个未提交的事务写的数据，被称为脏读。基本情景如下：

| 时间 |  事务A   |  事务B   |
| :--: | :------: | :------: |
|  1   | 开启事务 | 开启事务 |
|  2   | a = 100  | a = 100  |
|  3   |    /     | a = 110  |
|  4   | a = 110  |    /     |
|  5   |    /     | rollback |



## 不可重复读

事务A读取一个字段，然后事务B**更新**了字段，事务A再次读取发现两次值不相等。

| 时间 |  事务A   |  事务B   |
| :--: | :------: | :------: |
|  1   | 开启事务 | 开启事务 |
|      | a = 100  | a = 100  |
|  3   |    /     | a = 110  |
|  4   | a = 110  |  commit  |



## 幻读

事务A读取一个字段，然后事务B**插入**了满足条件的数据，事务A再次读取发现两次值不一样。

| 时间 |                    事务A                    |              事务B              |
| :--: | :-----------------------------------------: | :-----------------------------: |
|  1   |                  开启事务                   |            开启事务             |
|  2   | select * from table where id = 1(记录为0条) |                /                |
|  3   |                      /                      | insert into table (id) value(1) |
|  4   |                      /                      |             commit              |
|  5   |       insert into table (id) value(1)       |                /                |
|  6   |           commit(报错，主键冲突)            |                /                |



## 事务隔离级别

由于数据库在并发事务中带来一些问题，数据库提供了事务隔离机制来解决相对应的问题。数据库的锁也是为了构建这些隔离级别而存在的。

| 隔离级别                  | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） | 幻读（Phantom Read） |
| ------------------------- | ------------------ | -------------------------------- | -------------------- |
| 未提交读(Read uncommited) | 可能               | 可能                             | 可能                 |
| 已提交读(Read commited)   | 不可能             | 可能                             | 可能                 |
| 可重复读(Repeatable read) | 不可能             | 不可能                           | 可能                 |
| 可串行化(Serializable)    | 不可能             | 不可能                           | 不可能               |

- 未提交读(Read Uncommitted)：在该隔离级别下，所有事务都能看到其他未提交事务的执行结果，会导致脏读、不可重复读、幻读。
- 提交读(Read Committed)：一个事务只能看见已经提交事务所做的改变，不能避免不可重复读、幻读。
- 可重复读(Repeated Read)：**默认的隔离级别**，事务A读取到一个数据后，事务B进行了修改并提交，事务A再次读取该数据，还是读取到原来的内容。不能避免幻读问题。
- 可串行化(Serializable)：对于同一行记录，写会加写锁，读会加读锁，当出现读写冲突的时候，后面的事务必须等待前一个事务执行完成才能继续执行。



**隔离级别越高，数据库的并发性能越差。**



查询与修改隔离级别的SQL语句：

```
-- 查看系统隔离级别：
select @@global.tx_isolation;
-- 查看当前会话隔离级别
select @@tx_isolation;
-- 设置当前会话隔离级别
SET session TRANSACTION ISOLATION LEVEL serializable;
-- 设置全局系统隔离级别
SET GLOBAL TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```





# 事务日志

事务日志有两种：

- REDO LOG：重做日志，提供再写入操作、恢复提交事务修改的页操作，用来保证事务的持久性。是存储引擎层生成的日志，记录**物理页**的修改操作（包括页号、偏移量等），主要保证数据的可靠性。
- UNDO LOG：回滚日志，回滚行记录到某个特定版本，用来保证事务的原子性、一致性。是存储引擎层生成的日志，记录**逻辑操作**日志，用于**事务回滚**和**一致性非锁定读**。





## redo log

InnoDB以页为单位管理存储空间，读取页首先要将**磁盘**中的页读取到内存中的Buffer Pool再访问，所有的更新都先更新缓冲池数据，缓冲池中的**脏页**会以一定的频率被刷入磁盘（**checkpoint**机制），以此来抵消CPU和磁盘的差距。



### 为什么需要redo log

由于checkpoint机制是定时触发，当出现事务提交后，刚写完缓冲池，数据库宕机，那么会发生数据丢失，不符合**持久性**的要求。

解决思路：

- 每次提交的数据都进行刷盘操作，存在以下问题：
  - 修改量和刷盘工作量不成比例：修改1KB内容，刷盘16KB。
  - 随机IO刷新较慢：一个事务内修改的物理页不一定连续。
- 使用redo log进行记录，包括物理页、偏移量、修改值等数据。
  - 降低了频繁刷盘的效率。



InnoDB采取的是WAL（**Write-Ahead Logging**），先写日志，后写磁盘，发生宕机后通过redo log进行恢复，**保证持久性，这就是redo log的作用。**



### 特点

- redo日志**顺序写入磁盘**：磁盘顺序写的性能和写内存差不多。
- 事务执行过程中，redo日志不断记录。
  - bin log直到事务提交，才会一次性写入到bin log中。
- 占用空间比较小。





### 组成

主要分为两部分：

- redo log buffer：保存在内存中
  - innodb_log_buffer_size：默认16M
- redo log file：保存在磁盘中



### 整体流程

以更新事务为例，基本流程如下：

1. 先将原始数据从磁盘读取到内存中，并进行更新。
2. 生成一条redo log entry并写入redo log buffer。
3. 当事务commit后，将redo log buffer中的内容刷新到redo log file，采用append方式。
4. 定期将内存中修改的数据刷新到磁盘中。



### redo log buffer的刷盘策略

该流程是保证数据持久化的核心环节，这里的刷盘并不直接刷到磁盘，而实刷入**Page Cache（内存）**中（write），真正的刷盘工作（fsync）交给OS去做。如果此时系统宕机，那么数据会丢失。因此InnoDB提供了`innodb_flush_log_at_trx_commit`参数来控制何时将redo log buffer中的日志刷入redo log file。默认值为1。

- 0：redo log buffer每隔一秒将其数据刷入page cache，该模式下事务提交不会触发刷盘操作。
  - mysql进程崩溃会导致数据丢失。
- 1：每次事务提交都会将将redo log buffer中数据刷入page cache，并立刻刷入磁盘。效率较低也为安全。
- 2：每次事务提交都会将redo log buffer中数据刷入page cache，由OS同步到磁盘。（每秒一次）
  - mysql进程崩溃不会有数据丢失，当时OS宕机会有数据丢失。



### redo log buffer的写入过程



Mini-Transaction：MySQL将对底层页面中的一次原子访问的过程称为Mini-Transaction（mtr）。例如：向默认索引对应的B&#43; Tree中插入一条记录就是一次mtr。每个mtr会包含一组redo log entry，在进行恢复数据时，一组redo日志作为不可分割的整体。



写入是顺序写入，先往前面的block中写，当写满后，往后续的block写入，提供了`buf_free`的全局变量用来指明写到了哪个block。



一个block共512字节（磁盘扇区为512，避免非原子性写入），block由以下几个部分组成：

- log block header：12字节，保存一些block元数据。
- log block body：492字节，存储redo log信息。
- log block trailer：8字节，保存校验值。





### redo log file



相关配置参数：

- innodb_log_group_home_dir：指定redo log文件存储路径，默认为`./`，表示在数据库的数据目录中（默认为`var/lib/mysql`）
- innodb_log_files_in_group：指定redo log file的个数，命名方式如：ib_logfile0，默认为2，最大100
- innodb_log_file_size：单个redo log file的大小，默认为48M



采用循环使用的方式进行写入，整个日志组有两个重要的属性：

- write pos：当前记录的位置，一边写一边后移。
- checkpoint：当前要擦除的位置，也是往后推移。

每次刷盘redo log，write pos就往后更新，每次MySQL加载日志文件恢复数据，会清空加载过的日志，并将checkpoint后移更新，两个变量之间的部分用做空闲空间来写入新数据，类似于一个**环形队列**。

当两者相遇，表示文件组已满。



## undo log

undo log保证了事务的原子性，主要用来实现事务回滚操作。



### 为什么需要undo log

事务需要保证**原子性**，如果出现意外情况，例如mysql进程崩溃，OS宕机、断电，或者是事务本身`ROLLBACK`，都需要对数据进行**回滚**，因此使用undo log完成该项任务。



undo log会产生redo log，因为undo log也需要持久性的保护。



### undo log作用

- **回滚数据**：undo并不是**物理**恢复（数据页操作），undo log是**逻辑日志**，只是将数据库逻辑上恢复到原来的样子。
- **MVCC**：InnoDB中的MVCC通过undo来完成的，当用户读取一行记录时，若该纪录已被其他事务占用，当前事务可以通过undo读取之前的版本，以此实现**非锁定读**。



### undo存储结构

采用段的方式进行管理（回滚段），每个回滚段记录了1024个`undo log segment`，在每个`undo log segment`中进行undo页的写入。



回滚段和事务的关系：

- 每个事务只使用一个回滚段，一个回滚段在同一时刻服务于多个事务。
- 事务开始时，会指定一个回滚段，在事务执行过程中，当数据被修改，原始数据会复制到回滚段。
- 回滚段中事务不断填冲盘区，使用完会扩展下一个盘区，所有已分配的盘区都被用完，会覆盖最初的盘区。
- 事务提交时，存储引擎处理两个事情：
  - 将undo log放在列表中，以供后续的purge操作。
  - 判断undo log所在页是否可重用
    - 该页会被放入链表，并判断可用空间是否小于四分之三，小于则可以被重用，不被回收。



### 回滚数据分类

- 未提交的回滚数据：此时事务暂未提交，用于实现读一致性，所以该数据不能被其他事务的数据覆盖。
- 已提交但未过期的回滚数据：事务已提交，受`undo retention`参数的保持时间的影响。
- 事务已经提交并过期的回滚数据：数据保存时间超过`undo retention`指定的时间，回滚段满了后，优先覆盖此部分数据。



### undo log参数

- innodb_undo_directory：设置undo log存储路径，默认值为`./`。
- innodb_undo_logs：设置`rollback segment`的个数，默认为128
- innodb_undo_tablespaces：设置`rollback segment`文件的数量。默认为2。





### undo类型

InnoDB中undo log分为以下两种：

- insert undo log：在insert操作下产生的log，只对事务本身可见，对其他事务不可见（隔离性的要求），事务提交后直接删除，不需要purge。
- update undo log：delete和update操作产生的log，需要提供MVCC机制，因此不能直接删除，**等待purge线程进行删除。**



## ACID的实现



### 原子性的实现
每一个写事务，都会修改Buffer Pool，并产生对应的Redo日志，Redo日志以Write Ahead Log方式写，如果不写日志，数据库宕机恢复后，事务无法回滚，无法保证原子性。

### 持久性的实现

通过WAL可以保证逻辑上的持久性，物理上的持久性通过存储引擎的数据刷盘实现。

### 隔离性的实现

通常用Read View表示一个事务的可见性，读提交状态每一条读操作语句都会获得一次Read View，每次更新都会获取最新事务的提交状态，即每条语句执行都会更新其可见性视图。可重复读的隔离级别下，可见性视图只有在自己当前事务提交之后，才会更新，所以与其他事务没有关系。



**在MySQL中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，可以得到之前一个状态的值**。假设一个值从1被按照顺序改成了2，3，4，在回滚日志中会有类似下面的记录。



![记录](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/transaction/1.jpg)



当前值是4，但是在查询这条记录的时候，不同时刻启动的事务会有不同的`read-view`，在视图A、B、C中，记录的值分别为1、2、4。同一条记录在系统中可以存在多个版本，就是数据库的多版本并发控制（MVCC ）。对于`read-view A`要得到1，就必须将当前值依次执行图中所有的回滚操作得到。

即使现在有另外一个事务正在将 4 改成 5，这个事务跟 `read-view A、B、C `对应的事务是不会冲突的。



### 一致性的实现

一致性是通过其它三个特性来保证的。而其它三个特性由Redo、Undo来保证的。









# 锁



## 锁的分类

- 对数据的操作类型划分
  - 读锁/共享锁
  - 写锁/排他锁
- 粒度划分
  - 表级锁
  - 行级锁
  - 页级锁
- 对锁的态度划分
  - 悲观锁
  - 乐观锁
- 加锁方式
  - 显式锁
  - 隐式锁








## InnoDB的锁


InnoDB中，锁分为行锁和表锁，行锁包括两种锁。
- 共享锁（S）：共享锁锁定的资源可以被其它用户读取，但不能修改，在进行SELECT的时候，会将对象进行共享锁锁定，数据读取完毕，释放共享锁，保证在读取的过程中不被修改。
- 排他锁（X）：锁定的数据只允许进行锁定操作的事务使用，其它事务无法对已锁定的数据进行查询和修改

```SQL
# 给product_comment加上共享锁
LOCK TABLE product_comment READ;

# 解锁
UNLOCK TABLE;

# 对 user_id=912178 的数据行加上共享锁
SELECT comment_id, product_id, comment_text, user_id FROM product_comment WHERE user_id = 912178 LOCK IN SHARE MODE

# 给 product_comment 数据表添加排它锁
LOCK TABLE product_comment WRITE;

# 解锁
UNLOCK TABLE;

# 对 user_id=912178 的数据行加上排他锁
SELECT comment_id, product_id, comment_text, user_id FROM product_comment WHERE user_id = 912178 FOR UPDATE;
```





### 意向锁

InnoDB为了允许行锁和表锁共存，实现多粒度锁机制，意向锁就是其中的一种表锁。

- 意向锁的存在是为了协调行锁和表锁的关系
- 意向锁是一种不与行级锁冲突的表级锁
- 表明某个事物正在某些行持有了锁或该事务准备去持有锁。



InnoDB还有两种内部使用的意向锁，也都是表锁，表锁分为三种：

- 意向共享锁（IS）：事务计划给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁。
- 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。
- 自增锁（AUTO-INC Locks）：特殊表锁，自增长计数器通过该“锁”来获得子增长计数器最大的计数值。自增主键会涉及自增锁，在INSERT结束后立即释放。

- 如果事务想要获取数据表中某些数据的共享锁，就会在表上添加**意向共享锁**。
- 如果事务想要获取数据表中某些数据的排他锁，就会在表上添加**意向排他锁**。

意向锁主要为了**提高效率**，避免线程去逐个检查行锁。

在加行锁之前必须先获得表级意向锁，否则等`innodb_lock_wait_timeout` 超时后根据`innodb_rollback_on_timeout` 决定是否回滚事务。



插入数据的方式分为三类：

- Simple inserts：预先确定插入的行数，
  - `INSERT VALUES()`、`REPLACE`
- bulk inserts：事先不知道插入的行数，每处理一行，为AUTO_INCREMENT分配一个新值。
  - `INSERT ... SELECT`、`LOAD DATA`
  - mixed-mode inserts：是simple inserts语句，但是指定部分新行的自动递增值

AUTO-INC锁是当插入数据中含有`AUTO_INCREMENT`字段时进行lock的表级锁。并发性能较差。



InnoDB通过`innodb_autoinc_lock_mode`来提供不同的锁定机制。

- 0：所有insert语句都会获得同一个AUTO-INC锁
- 1：bulk inserts仍使用AUTO-INC锁，Simple inserts由于确定插入条数，在获取自增锁后会释放
- 2：所有insert语句都不会使用表级AUTO-INC锁，自增值保证所有insert语句获得的值是唯一的，由于多个语句同时生成，生成值可能不是连续的。

锁关系矩阵如下图所示：

![InnoDB锁关系矩阵](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/transaction/%E9%94%81%E5%85%BC%E5%AE%B9%E7%9F%A9%E9%98%B5.png)



### 元数据锁

MySQL5.5引入了meta data lock，简称MDL锁（表锁），其作用是保证读写的正确性。当对一个表做增删改查操作时，加MDL读锁，当对表结构做变更操作时，加MDL写锁。



### 行锁

InnoDB行锁是通过对索引数据页上的记录加锁实现的，在存储引擎层实现。

主要有三种锁：

- Record锁（LOCK_REC_NOT_GAP）：单个行记录的锁（锁数据，不锁Gap）
- Gap锁：间隙锁，锁定一个范围，不包括记录本身（不锁数据，锁数据前面的Gap）
  - 为了防止插入幻影记录而提出的

- Next-key锁：锁数据并且锁Gap，可以解决幻读问题。
  - **既想锁住某条记录，又想阻止其他事务在该记录前面的间隙插入新纪录。**

- 插入意向锁（Insert Intention Locks）：插入一条记录需要判断是否存在gap锁，有的话需要等待。InnoDB规定事务在等待的时候也需要在内存中生成一个**锁结构**，**表明有事务想在某个间隙中插入新记录**，这个锁结构被称为`Insert Intention Locks`。
  - 插入意向锁并不会阻止别的事务继续获取该记录上任何类型的锁。



### 全局锁

对**整个数据库实例**加锁，整个数据库处于**只读状态**，，使用场景为：**全库逻辑备份**。

```sql
Flush tables with read lock
```






### InnoDB死锁

由于InnoDB是逐行加锁的，极容易产生死锁，产生死锁的四个条件：
- 互斥条件：一个资源每次只能被一个进程使用。
- 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
- 不剥夺条件：进程以获得的资源，在没有使用完之前，不能强行剥夺。
- 循环等待条件：多个进程之前形成的一种互相循环等待资源的关系。



### InnoDB中如何处理死锁

- 等待，直到超时：`innodb_lock_wait_timeout`参数进行控制，默认为50s，超时回对事务进行**回滚**。
- 使用死锁检测进行处理：InnoDB使用`wait-for graph`算法来主动进行死锁检测，每次加锁请求无法立即满足时，都会触发此算法。
  - 这是一种主动的检测机制，要求数据库保存**锁的信息链表**和**事务等待链表**两部分信息。
  - 基于这两部分信息，绘制`wait-for graph`
  - 一旦检测到回路，引擎会选择回滚**undo量最小的事务**，让其他食物继续执行。（此部分通过`innodb_deadlock_detect`参数控制）
- 



### 避免死锁

- 更新SQL的where条件尽量用索引
- 合理设置索引，加锁索引准确，缩小锁定范围，减少锁竞争。
- 减少范围更新，尤其非主键/非唯一索引的范围更新。
- 控制事务大小，减少锁定数据量和锁定时间长度
- 加锁顺序一致，尽可能一次性锁定所有所需的数据行。
- 并发要求高的系统，不要显式加锁。





### 锁的结构

如果出现以下情况，多个逻辑上的锁会放在一个锁结构中：

- 同一个事务中进行加锁
- 被加锁的记录在同一个页面
- 加锁的类型是一样的
- 等待状态一样



锁结构包含以下字段：

- 锁所在事务信息：记录哪个事务生成了这个锁结构，指针。
- 索引信息：对于行锁，需要记录加锁的记录属于哪个索引，指针。
- 表锁/行锁信息：用于区分此类型。
  - 表锁记录：表信息、其他信息
  - 行锁记录：
    - Space ID：所在的表空间
    - Page Number：记录所在页号
    - n_bits：一条记录对应一个bit位，用不同的bit位区分哪一条记录加了锁。
- type_mode：32bit，分为以下几个部分
  - lock_mode：低4位。
    - LOCK_IS （十进制的 0 ）：表示共享意向锁，也就是IS锁 。 
    - LOCK_IX （十进制的 1 ）：表示独占意向锁，也就是IX锁 。
    -  LOCK_S （十进制的 2 ）：表示共享锁，也就是S锁 。
    -  LOCK_X （十进制的 3 ）：表示独占锁，也就是X锁 
    -  LOCK_AUTO_INC （十进制的 4 ）：表示 AUTO-INC锁 。
  - lock_type：低5-8位，目前只有第5、6位被使用
    - LOCK_TABLE （十进制的 16 ），也就是当第5个比特位置为1时，表示表级锁。 
    - LOCK_REC （十进制的 32 ），也就是当第6个比特位置为1时，表示行级锁。
  - rec_lock_type：其余位。
    - LOCK_ORDINARY （十进制的 0 ）：表示 next-key锁 。 
    - LOCK_GAP （十进制的 512 ）：也就是当第10个比特位置为1时，表示 gap锁 。 
    - LOCK_REC_NOT_GAP （十进制的 1024 ）：也就是当第11个比特位置为1时，表示记录锁 。 
    - LOCK_INSERT_INTENTION （十进制的 2048 ）：也就是当第12个比特位置为1时，表示插入意向锁。



### 锁监控

可以检查`InnoDB_row_lock`变量来分析锁状态。

```sql
show status like &#39;innodb_row_lock%&#39;;

Variable_name 	 				Value
Innodb_row_lock_current_waits	0
Innodb_row_lock_time			297
Innodb_row_lock_time_avg		42
Innodb_row_lock_time_max		83
Innodb_row_lock_waits			7
```

各字段含义分别如下：

- Innodb_row_lock_current_waits：当前正在等待锁定的数量； 
- Innodb_row_lock_time ：从系统启动到现在锁定总时间长度；（等待总时长） 
- Innodb_row_lock_time_avg ：每次等待所花平均时间；（等待平均时长）
- Innodb_row_lock_time_max：从系统启动到现在等待最常的一次所花的时间；
- Innodb_row_lock_waits ：系统启动后到现在总共等待的次数；（等待总次数）



MySQL把事务和锁的信息记录在了 information_schema 库中，涉及到的三张表分别是 `INNODB_TRX` 、 `data_locks`（8.0更新） 和 `data_lock_waits`（8.0更新） 。







# 并发控制



## 单版本控制-锁

锁用独占的方式保证只有一个版本的情况下事务相互隔离。



## 多版本控制-MVCC

MVCC（Multi-Version Concurrency Control）即多版本并发控制。

MVCC 是通过保存数据在**某个时间点的快照**来实现并发控制的。简单来说它的思想就是**保存数据的历史版本**。这样我们就可以通过比较版本号决定数据是否显示出来。读取数据的时候不需要加锁也可以保证事务的隔离效果。

每次对数据库的修改，都会在Undo日志中记录当前修改记录的**事务版本号**以及**修改前数据状态的存储地址**，以便在必要的时候可以回滚到老的数据版本。

MVCC的实现依赖于：Undo Log、Read View、隐藏字段。



MVCC没有正式的标准，在不同DBMS中实现方式也不同。



### 解决的问题

MVCC主要解决以下几个问题：

- 读写之间阻塞的问题，通过MVCC可以让读写互相不阻塞，提高并发处理能力。
- 降低了死锁的概率，MVCC采用了乐观锁的方式，读取数据时不需要加锁，对于写操作，只锁定必要的行。
- 解决一致性读的问题。一致性读也被称为快照读，当我们查询数据库在某个时间点的快照时，只能看到这个时间点之前事务提交更新的结果，而不能看到这个时间点之后事务提交的更新结果。


MVCC最大的**好处**是**读不加锁，读写不冲突**，在MVCC中，读操作可分为快照读（Snapshot Read）和当前读（Current Read）。 

- 快照读：读取的是记录的可见版本，不用加锁
- 当前读：读取的是记录的最新版本，并且当前读返回的记录，都会加锁，保证其它事务不会并发修改这条记录。

```SQL
# 不加锁的SELECT都是快照读
SELECT * FROM player WHERE ...

# 加锁的SELECT和增删改都是用当前读
SELECT * FROM player LOCK IN SHARE MODE;
SELECT * FROM player FOR UPDATE;
INSERT INTO player values ...
```

**MVCC只在`Read Commited`和`Repeatable Read`两种级别下工作。核心是Undo Log &#43; Read View。**





### Read VIew

在 MVCC 机制中，多个事务对同一个行记录进行更新会产生多个历史快照，这些历史快照保存在 Undo Log 里。而Read View保存了不应该让这个事务看到的其他的事务ID列表。



Read View就是事务使用MVCC机制进行快照读操作时产生的读视图，事务启动时，会产生数据库系统当前的一个快照。



**主要问题**：判断版本链中哪个版本是当前事务可见的。



ReadView主要包含以下内容：

- creator_trx_id：创建这个Read View的事务ID。
- trx_ids：生成Read View时系统中活跃的读写事务的**事务id列表**。
- up_limit_id：活跃的事务中最小的事务ID。
- low_limit_id：生成Read View时系统中应该分配给下一个事务的id





### ReadView规则

根据以下步骤判断记录中某个版本是否可见：

- 如果被访问版本的`trx_id = ReadView 中的 creator_trx_id ` ，表示当前事务在访问它修改过的记录，所以该版本可以被当前事务访问。
- 如果被访问版本的`trx_id &lt; ReadView 中的 creator_trx_id `，表明生成该版本的事务已经提交，所以该版本可以被当前事务访问。
- 如果被访问版本的trx_id属性值 &gt;= ReadView中的 low_limit_id 值，表明生成该版本的事 务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。
- 如果被访问版本的trx_id属性值在ReadView的 up_limit_id 和 low_limit_id 之间，那就需要判断一下trx_id属性值是不是在 trx_ids 列表中。
  - 在其中，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问。
  - 不在其中，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。



### MVCC流程

查询一条记录时，MVCC工作流程如下：

1. 首先获取事务的`trx_id`
2. 获取对应的ReadView
3. 查询读到的数据，与ReadView中的`trx_id`进行比较
4. 如果不符合上述规则，从`Undo Log`中获取历史快照。
5. 返回最终符合规则的数据。



**MySQL怎么解决脏读、不可重复读、幻读等问题呢？**

- 读操作使用MVCC，写操作进行加锁

MVCC会生成一个**ReadView**，通过`ReadView`找到符合条件的记录版本，查询语句只能读到在生成ReadView之前**已提交事务做的更改**。

在不同隔离级别下使用MVCC会有不同结果：

- `READ COMMITED`：每次执行SELECT操作都会生成一个ReadView，ReadView本身就保证了只能读到已提交事务的数据，避免了脏读的现象，但是会有不可重复读和幻读问题。
- `REPEATABLE READ`：一个事务在执行过程中，只有**第一次执行SELECT操作**才会生成一个ReadView，之后的读操作**复用**这个ReadView，避免了不可重复读和幻读的问题。
  - 此隔离级别下，**快照读直接通过MVCC即可解决。**
  - **当前读通过`Next-key Lock`来锁定本记录和索引区间防止插入情况，从而避免幻读。**









---

> Author:   
> URL: http://localhost:1313/posts/mysql/%E6%8E%A2%E7%B4%A2mysql%E7%9A%84%E4%BA%8B%E5%8A%A1%E4%B8%8E%E9%94%81%E6%9C%BA%E5%88%B6/  

