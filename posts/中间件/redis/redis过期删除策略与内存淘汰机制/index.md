# Redis过期删除策略与内存淘汰机制


## 过期删除策略



过期删除策略主要有以下三种：

- 定时删除：设置过期时间的同时，创建一个定时器，当到达过期时间后，执行删除操作。
  - 优点：对内存友好，不会长时间占用内存
  - 缺点：对CPU不友好，当过期entry过多会占用大量CPU时间，并且创建定时器存在性能消耗。
- 惰性删除：每次查询到该entry时，检查是否过期，若过期就删除，否则返回。
  - 优点：对CPU友好
  - 缺点：无用数据占用大量内存空间，依赖于客户端请求对过期数据进行删除。有内存泄漏的风险。
- 定期删除：每隔一段时间对数据库进行检查，删除其中的过期entry。
  - 综合考虑上述策略的优缺点，可以合理设置执行时长和频率。



Redis中主要配合使用**惰性删除**和**定期删除**两种策略，在内存和CPU性能中取得平衡。
使用过期字典存储所有key的过期时间。key是一个指针，指向key对象，value是一个long long类型的整数，保存key的过期时间。


实现：

- 惰性删除：`db.c/expireIfNeeded`在读写之前对key进行检查.
- 定期删除：`redis.c/activeExpireCycle`实现，每当Redis的`serverCron`执行时，都会主动清除过期数据。
  - `activeExpireCycle`的工作流程如下：
    - 每次取出一定数量的随机key进行检查，并删除其中的过期数据
    - 全局变量`current_db`存储当前的检查进度（db编号），并且下次`activeExpireCycle`执行会接着上次进度进行处理。全部检查完毕后该变量置为0。





## 内存淘汰策略
当Redis的运行内存已经超过设置的最大内存时，会使用内存淘汰策略删除符合相关条件的key。
最大内存通过设置`maxmemory &lt;bytes&gt;`即可。默认为0，表示没有内存大小的限制。

Redis的内存淘汰策略如下：

- 不淘汰
  - noeviction：不淘汰任何数据、直接返回错误。


- 对设置了过期时间的数据进行淘汰
  - volatile-random： 随意淘汰设置了过期时间的任意entry。
  - volatile-ttl：优先淘汰更早过期的entry。
  - volatile-lru：淘汰所有设置了过期时间的entry中最近最久未使用的entry。
  - volatile-lfu：淘汰所有设置了过期时间的entry中最少使用的entry。

- 对全部数据进行淘汰
  - allkeys-random：随即淘汰任意entry。
  - allkeys-lru：使用LRU策略淘汰任意entry。
  - allkeys-lfu：使用LFU策略淘汰任意entry。


修改redis.conf中的`maxmemory-policy &lt;策略&gt;`即可。


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis%E8%BF%87%E6%9C%9F%E5%88%A0%E9%99%A4%E7%AD%96%E7%95%A5%E4%B8%8E%E5%86%85%E5%AD%98%E6%B7%98%E6%B1%B0%E6%9C%BA%E5%88%B6/  

