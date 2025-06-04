# Redis基础知识总结


## Redis简介

### Redis特性
- 速度快（内存，10w QPS, C , 单线程）
- 持久化（将内存数据异步更新到磁盘,RDB&amp;AOF）
- 多种数据结构（string list set zset hash BitMaps HyperLogLog GEO）
- 支持多语言
- 功能丰富（发布订阅 事务 Lua脚本 pipeline）
- 简单（23000 lines of code 不依赖外部库 单线程模型）
- 主从复制 

单线程为什么这么快？
1. 纯内存
2. 非阻塞IO，使用多路复用机制，提升连接并发度。
3. 避免多线程切换和竞态消耗



### Redis典型应用场景
- 缓存系统（缓存）
- 计数器（微博转发和评论）
- 消息队列系统
- 排行榜
- 社交网络
- 实时系统（垃圾邮件--布隆过滤器）

### Redis可执行文件说明

```
redis-server      ---&gt;    Redis服务器
redis-cli         ---&gt;    Redis客户端
redis-benchmark   ---&gt;    Redis性能测试
redis-check-aof   ---&gt;    AOF文件修复
redis-check-dump  ---&gt;    RDB文件检查
redis-sentinel    ---&gt;    Sentinel服务器
```


### Redis 通用API


```
keys                ---&gt;    遍历所有key，热备从节点         O(n)
dbsize              ---&gt;    计算key的总数                   O(1)
exist key           ---&gt;    检查key是否存在，返回0或1       O(1)
del key             ---&gt;    删除指定的key                   O(1)
expire key seconds  ---&gt;    key在seconds秒后过期            O(1)         
ttl key             ---&gt;    查看key的剩余过期时间           O(1) 
persist key         ---&gt;    去掉key的过期时间               O(1) 
type key            ---&gt;    查看当前key的类型               O(1)

```



## 常见数据结构

String应用场景：字符串缓存、计数器、分布式锁

常用API：
```
get key
del key
incr                            整型自增
decr                            整型自减
incrby key k                    key自增k，如果key不存在，自增后get(key) = k
decr key k                      key自减k，如果key不存在，自减后get(key) = k
set key value                   无论是否存在
setnx key value                 key不存在才能设置
set key value xx                key存在，才设置
mget key1 key2                  批量获取
mset key1 value1 key2 value2    批量设置
getset key newvalue             set key newvalue并返回旧的value
append key value                将value追加到旧的value中
strlen key                      返回字符串长度
incrbyfloat key 3.5             增加key对应的值3.5
getrange key start end          获取字符串指定下标所有的值
setrange key index value        设置指定下标所有对应的值
```

Hash应用场景：存储键值对数据

常用API：
```
hget key field                  获取hash key对应的field的value
hset key field value            设置hash key对应field的value
hdel key field                  删除
hexists key field               判断hash key是否有field                        
hlen key                        获取hash key field长度
hmget key field1 field2         批量获取
hmset key1 field1 value1        批量设置
hgetall key                     返回hash key对应的所有的field和value
hvals key                       返回hash key对应所有的field的value
hkeys key                       返回hash key对应的所有field
hsetnx key field value          设置hash key对应的field的value
hincrby key field intCounter    hash key 对应的field的value自增intCounter
hincrbyfloat
```


List特点：可重复的、可左右弹出压入的

常用API：
```
rpush key value1 value2                     从列表右端进行插入
lpush
linsert key before|after value newValue     在list指定的前后newValue
lpop key                                    左边弹出一个元素
rpop key                                    右边弹出一个元素
lrem key count value                        根据count值，删除所有value相等的项，count&gt;0 从左往右删除。count&lt;0从右往左，count=0删除所有value相等的值
ltrim key start end                         按照索引范围修剪列表
lrange key start end                        获取列表指定所有范围的item
lindex key index                            获取列表指定索引的item
llen key                                    获取列表长度
lset key index newValue                     设置指定索引为newValue
blpop key timeout                           lpop阻塞版本 timeout是阻塞超时时间
brpop                                       rpop阻塞版本
```



Set特点：无序、不重复

常用API：
```
sadd key element            向集合key中添加element
srem key element            将集合中的element移出
scard key                   计算集合大小
sismember key element       判断是否在集合中
srandmember key count       从集合中随机挑出count个元素
smembers key                获取集合所有元素
spop key                    随机弹出一个元素
sdiff key1 key2             差集
sinter key1 key2            交集
sunion key1 key2            并集
```

zset特点：有序、无重复元素

常用API：
```
zadd key score element              添加score和element  O(logN)
zrem key element                    删除元素            O(1)
zscore key element                  返回元素的分数
zincrby key increScore element      增加或者减少元素的分数
zcard key                           返回元素的总个数
zrank
zrange key start end                返回索引内的元素
zrangebyscore key minScore maxScore 返回指定分数范围内的升序元素
zcount key minScore maxScore        返回有序集合内指定范围内的个数
zremrangebyrank key start end       删除指定排名内的升序元素
zremrangebyscore key min max        删除指定分数范围内的升序元素

zrevrank                            
zrevrange
zrevrangebyscore
zinterstore
zunionstore
```



### Bitmap

位图：字符以二进制格式存储在内存中

常用API：
```
setbit key offset value                 给位图指定索引设置值
getbit key offset
bitcount key [start end]                获取位值为1的个数
bitop op destkey key                    做多个bitmap的and or not xor操作并将结果保存到destKey中
bitpos key targetBit [start] [end]      计算位图指定范围第一个偏移量对应的值等于targetBit的位置

```



### HyperLogLog

特点如下：
- 基于HyperLogLog算法：极小空间完成独立数量统计
- 本质还是string

常用API：
```
pfadd key element                       添加元素
pfcount key                             计算独立总数
pfmerge destKey sourcekey [sourcekey]   合并多个hyperloglog

```


### GEO

GEO：存储经纬度，计算两地距离，范围计算

常用API：
```
geoadd key longitude latitude member                   增加地理位置信息
geoadd cities:locations 116.28 39.55 beijing
geopos key member                                       获取地理位置信息
geodist key member1 member2 [unit]                      获取两地位置的距离，unit：m,km,mi,ft
georadius
```

- since 3.2 &#43;
- type geoKey = zset
- 没有删除API：zrem key member



## 其他功能

### 慢查询

慢查询日志就是系统在命令执行前后计算每条命令的执行时间，当超过预设阈值，就将这条命令的相关信息记录下来，以便后期的问题排查和调优。


```

默认配置参数
slowlog-max-len = 128                  # 最多能保存的慢查询日志条数，若超过采用FIFO方案将最旧的慢查询日志先删除。
slowlog-log-slower-than = 10000        # 慢查询的预设阀值（单位：微秒）

如果slowlog-log-slower-than的值是0，则会记录所有命令。
如果slowlog-log-slower-than的值小于0，则任何命令都不会记录日志。

慢查询命令

slowlog get [n]         获取慢查询队列
slowlog len             获取慢查询队列长度
slowlog reset           清空慢查询队列
```



### Pipeline

客户端将多个请求打包，同时发给服务端。比较是否非实时需求，统一打包一次发送，减少网络传输的开销和时延问题。

### 事务
事务将一种或多种请求打包，然后一次性按顺序 执行多个命令的机制，事务执行过程中，服务器不会中断事务去执行其他请求。事务以`MULTI`命令开始，接着将多个命令放入事务中，由`EXEC`将这个事务提交给服务器。




### 发布订阅模式

主要有以下角色：
- 发布者（publisher）
- 订阅者（subscriber）
- 频道（channel）

常用API
```
publish channel message
subscribe channel
unsubscribe channel
psubscribe [pattern]    订阅模式
punsubscribe [pattern]  退出指定的模式
pubsub channels         列出至少有一个订阅者的频道
pubsub numsub [channel] 列出指定频道的订阅者数量
pubsub numpat           列出被订阅者的数量
```




## 缓存的使用和设计

### 缓存的收益和成本

收益：
- 加速读写
- 降低后端负载



成本：
- 数据不一致：缓存层和数据层由时间窗口不一致，和更新策略有关
- 代码维护成本：多了一层缓存逻辑
- 运维成本



使用场景：
1. 降低后端负载
    - 对高消耗的SQL：join结果集、分组统计结果缓存
2. 加速请求响应
    - 利用Redis优化IO响应时间
3. 大量写合并为批量写
    - 计数器先Redis累加再批量写DB



### 缓存的更新策略
1. LRU/LFU/FIFO算法剔除，例如`maxmemory=policy`
2. 超时剔除，例如`expire`
3. 主动更新：开发控制生命周期




### 缓存的粒度控制
- 通用性：全量属性更好
- 占用空间：部分属性更好
- 代码维护：表面上全量属性更好


## 缓存问题

### 缓存穿透

cache和数据库中都没有的数据，但是用户不断发起请求。流量大的时候，DB可能宕机。



原因：

1. 业务代码自身问题
2. 恶意攻击，爬虫



#### 解决方法
1. 增加Filter，增加参数合法校验
2. 使用布隆过滤器，过滤Redis请求
3. 缓存空对象


```
public String getPassThrought(String key) {
    String cacheValue = cache.get(key);
    if(StringUtils.isBlank(cacheValue)){
        String storageValue = storage.get(key);
        cache.set(key,storageValue);
        //如果存储数据为空，设置一个过期时间
        if(StringUtils.isBlank(storageValue)){
            cache.expire(key,60 * 5);
        }
        return storageValue;
    }else {
        return cacheValue;
    }
}
```





### 缓存击穿

cache中没有数据，但是数据库中有数据，某个热点Key突然失效，导致大量请求都打在DB中。

#### 解决方法
- 根据实际场景，考虑热点数据是否可以设置为永不过期。
- 接口限流、服务熔断、降级来保护服务。






### 缓存雪崩

某时刻大量Key同时失效，流量直接打到DB，导致DB流量瞬间加大，造成级联故障。

#### 解决方法
1. 保证缓存高可用性
    - 个别节点，个别机器
    - Redis Cluster，Redis sentinel
2. key的失效时间设置为随机值，避免使用相同的过期时间。
3. 依赖隔离组件为后端限流，考虑增加熔断机制。
4. 压力测试提前发现问题。








### 缓存无底洞问题

2010年，FaceBook有3000个Memcache节点，发现问题：加机器没有性能提升，反而下降

问题关键点：
- 更多的节点 != 更高的性能
- 批量接口需求（mget,mset）
- 数据增长和水平扩展需求

#### 优化IO的几种方法
1. 命令本身优化，例如慢查询`keys`,`hgetall bigkey`
2. 减少网络通信次数
3. 减少接入成本：例如客户端长连接/连接池,NIO等

#### 四种批量优化的方法
1. 串行mget
2. 串行IO
3. 并行IO
4. hash_tag






## Redis安全

1. 设置密码
	- 服务端配置：requirepass和masterauth
	- 客服端连接：auth命令和-a参数
2. 伪装危险命令
  - 服务端配置：rename-command为空或者随机字符
  - 客户端连接：不可用或者使用指定随机字符
3. bind：服务端配置：bind限制的是网卡，并不是客户端IP
  - bind不支持config set
  - bind 127.0.0.1要谨慎
4. 防火墙
5. 定期备份
6. 不使用默认端口
7. 使用非root用户启动







---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86%E6%80%BB%E7%BB%93/  

