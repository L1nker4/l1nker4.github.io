# MySQL体系结构和存储引擎


## 体系结构

MySQL体系结构如图所示：
![MySQL体系结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/MySQL%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84.png)



分别由Client Connectors层、MySQL Server层以及存储引擎层组成。

- Client Connectors层：负责处理客户端的连接请求，与客户端创建连接。
- MySQL Server层：
    - Connection Pool：负责处理和存储数据库与客户端创建的连接，一个线程负责管理一个连接，包括了用户认证模块，就是用户登录身份的认证和鉴权以及安全管理
    - Service &amp; utilities：管理服务&amp;工具集，包括备份恢复、安全管理、集群管理、工具
    - SQL interface：负责接受客户端发送的各种语句
    - Parser：对SQL语句进行语法解析生成解析树
    - Optimizer：查询优化器会根据解析树生成执行计划，并选择合适的索引，然后按照执行计划执行SQL并与各个存储引擎交互
    - Caches：包括各个存储引擎的缓存部分，例如InnoDB的Buffer Pool
- 存储引擎层：包括InnoDB，MyISAM以及支持归档的Archive和内存的Memory
- 存储引擎底部是物理存储层，包括二进制日志，数据文件，错误日志，慢查询日志，全日志，redo/undo日志



一条SQL语句的执行过程可以参照如下图示：

![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/SQL%20process.png)

1. 与MySQL建立连接。
2. 查询缓存，如果开启了Query。 Cache并且查询缓存中存在该查询语句，则直接将结果返回到客户端，没有开启或缓存未命中则由解析器进行语法语义解析，并生成解析树。
3. 预处理器生成新的解析树。
4. 查询优化器进行优化。
5. 查询执行引擎执行SQL，通过API接口查询物理存储层的数据，并返回结果。

其中，查询缓存于MySQL 8.0中移除，具体原因：查询缓存往往弊大于利。查询缓存的失效非常频繁，只要有对一个表的更新，这个表上的所有的查询缓存都会被清空。

MySQL官方博客关于该技术移除的解释[https://mysqlserverteam.com/mysql-8-0-retiring-support-for-the-query-cache/](https://mysqlserverteam.com/mysql-8-0-retiring-support-for-the-query-cache/)



## 存储引擎

![MySQL存储引擎](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E.png)

在 MySQL 5.6 版本之前，默认的存储引擎都是 MyISAM，但 5.6 版本以后默认的存储引擎就是 InnoDB。

![InnoDB结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/InnoDB%E7%BB%93%E6%9E%84.png)

InnoDB上半部分是实例层，位于内存中，下半部分是物理层，位于文件系统中。
其中实例层分为线程和内存，InnoDB中重要的线程有Master Thread（主线程），其优先级最高，主要负责调度其他线程，其内部有几个循环：主循环，后台循环，刷新循环，暂停循环，Master Thread 会根据其内部运行的相关状态在各循环间进行切换。

大部分操作在主循环中完成，其包含1s和10s两种操作：



- 1s操作
    - 日志缓冲刷新到磁盘（即使事务未提交，也被执行）
    - 最多可以刷100个新脏页到磁盘
    - 执行并改变缓冲的操作
    - 若当前没有用户活动，可以切换到后台循环
- 10s操作
    - 最多可以刷新100个脏页到磁盘
    - 合并至多5个被改变的缓冲
    - 日志缓冲刷新到磁盘
    - 删除无用的Undo页
    - 刷新100个或10个脏页到磁盘，产生一个检查点
    - buf_dump_thread 负责将 buffer pool 中的内容 dump 到物理文件中，以便再次启动 MySQL 时，可以快速加热数据。
    - page_cleaner_thread 负责将 buffer pool 中的脏页刷新到磁盘，在 5.6 版本之前没有这个线程，刷新操作都是由主线程完成的，所以在刷新脏页时会非常影响 MySQL 的处理能力，在5.7 版本之后可以通过参数设置开启多个 page_cleaner_thread。
    - purge_thread 负责将不再使用的 Undo 日志进行回收。
    - read_thread 处理用户的读请求，并负责将数据页从磁盘上读取出来，可以通过参数设置线程数量。
    - write_thread 负责将数据页从缓冲区写入磁盘，也可以通过参数设置线程数量，page_cleaner 线程发起刷脏页操作后 write_thread 就开始工作了。
    - redo_log_thread 负责把日志缓冲中的内容刷新到 Redo log 文件中。
    - insert_buffer_thread 负责把 Insert Buffer 中的内容刷新到磁盘。实例层的内存部分主要包含 InnoDB Buffer Pool，这里包含 InnoDB 最重要的缓存内容。数据和索引页、undo 页、insert buffer 页、自适应 Hash 索引页、数据字典页和锁信息等。additional memory pool 后续已不再使用。Redo buffer 里存储数据修改所产生的 Redo log。double write buffer 是 double write 所需的 buffer，主要解决由于宕机引起的物理写入操作中断，数据页不完整的问题。


物理层在逻辑上分为系统表空间、用户表空间和Redo日志。

- 系统表空间有ibdata文件和一些Undo，ibdata文件有insert buffer段、double write段、回滚段、索引段、数据字典段和Undo信息段。
- 用户表空间之以`.ibd`后缀结尾的文件，文件中包含 insert buffer 的 bitmap 页、叶子页（这里存储真正的用户数据）、非叶子页。InnoDB 表是索引组织表，采用 B&#43; 树组织存储，数据都存储在叶子节点中，分支节点（即非叶子页）存储索引分支查找的数据值。
- Redo日志包括多个Redo文件，这些文件循环使用。当达到一定存储阈值会触发checkpoint刷脏页操作，同时也会在MySQL实例异常宕机后重启，InnoDB表数据自动还原回复过程中使用。

InnoDB内存结构：

![内存结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/InnoDB%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84.png)

- 用户读取或者写入的最新数据都存储在 Buffer Pool 中，如果 Buffer Pool 中没有找到则会读取物理文件进行查找，之后存储到 Buffer Pool 中并返回给 MySQL Server。Buffer Pool 采用LRU 机制。
- Redo log 是一个循环复用的文件集，负责记录InnoDB中所有对 Buffer Pool的物理修改日志






 InnoDB和MyIASM的区别

![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/InnoDB%20VS%20MyIASM.png)

![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/InnoDB.png)

其次，InnoDB性能优于MyIASM，CPU核数与InnoDB读写能力呈线性关系。



InnoDB 核心要点：

![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/structure/InnoDB%E8%A6%81%E7%82%B9.png)

---

> Author:   
> URL: http://localhost:1313/posts/mysql/mysql-structure-and-engine/  

