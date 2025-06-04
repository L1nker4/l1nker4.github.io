# Redis内存优化

## 查看内存使用情况

&gt; info memory

```
used_memory:701520                      redis分配器分配的内存量，也就是实际存储数据的内润使用总量
used_memory_human:685.08K               以可读格式返回redis使用的内存总量
used_memory_rss:664584                  从操作系统角度，redis进程占用的总物理内存
used_memory_rss_human:649.01K          
used_memory_peak:778480                  内存分配器分配的最大内存，代表`used_memory`的历史峰值
used_memory_peak_human:760.23K          以可读的格式显示内存的消耗峰值
total_system_memory:0
total_system_memory_human:0B
used_memory_lua:37888                   Lua引擎消耗的内存
used_memory_lua_human:37.00K
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
mem_fragmentation_ratio:0.95            内存碎片率，used_memory_rss/used_memory
mem_allocator:jemalloc-3.6.0            redis使用的内存分配器

```


## 内存划分

Redis中使用的内存可分为如下几类：
- 自身内存：自身维护的数据字典等元数据，占用较少
- 对象内存：存储的所有entry对象
- 缓存：客户端缓冲区&#43;AOF缓冲区等
- Lua内存：用于加载Lua脚本
- 子进程内存：一般是持久化时fork出来的子进程占用的内存
- 运行内存：运行时消耗的内存
- 内存碎片：Redis使用jemalloc来分配内存，按照固定大小划分，当删除数据后，释放的内存不会立即返回给OS，Redis无法有效利用，因此形成碎片。


## 内存优化方案

### 对象内存

Redis中对象大多有两种存储方案，尽量将大小控制在较为节省的存储阈值之内。


### 客户端缓冲区
客户端缓冲区占用内存较大的原因如下：
- client访问大key，导致client输出缓存异常增长。
- client使用monitor命令，它会将所有访问redis的命令持续放到输出缓冲区，导致输出缓冲区异常增长。
- clinet使用pipeline封装了大量命令
- 从节点复制慢，导致主节点输出缓冲区积压。

优化方案:
- 大key需要进行拆分
- 尽量避免使用monitor等指令
- 使用pipeline设置最大阈值
- 主从复制区设置阈值


### 内存碎片

- 手动执行memory purge命令
- 通过配置参数进行控制

```
activedefrag yes：启用自动碎片清理开关
active-defrag-ignore-bytes 100mb：内存碎片空间达到多少才开启碎片整理
active-defrag-threshold-lower 10：碎片率达到百分之多少才开启碎片整理
active-defrag-threshold-upper 100：内存碎片率超过多少，则尽最大努力整理（占用最大资源去做碎片整理）
active-defrag-cycle-min 25：内存自动整理占用资源最小百分比
active-defrag-cycle-max 75：内存自动整理占用资源最大百分比
```

### 子进程优化
Linux中的fork使用了copy on write机制，fork之后与父进程共享内存空间，只有发生写操作修改内存数据时，才会真正分配内存空间。


## 其他注意事项

### 键值生命周期

- 周期数据需要设置过期时间，`object idle time`可以找垃圾key-value
- 过期时间不宜集中，会导致缓存穿透和雪崩等问题

### 命令使用技巧
- O(n)以上命令关注n的数量
    - hgetall,lrange,smembers,zrange,sinter等 
- 禁用命令
    - 禁止线上使用keys,flushall,flushdb，通过redis的rename机制禁用掉命令，或者使用scan的方式渐进式处理 
- 合理使用select
    - redis的多数据库较弱，使用数字进行区分
    - 很多客户端支持较差
    - 同时多业务用多数据库实际上还是单线程处理，会有干扰
- redis的事务功能较弱，不建议过多使用
    - 不支持回滚
- redis集群版本在使用Lua上有特殊要求
    - 所有的key，必须爱一个slot上，否则返回error 
- 必要情况下使用monitor命令时，注意不要长时间使用



---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis%E5%86%85%E5%AD%98%E4%BC%98%E5%8C%96/  

