# Redis对象机制分析


## 简介

Redis中提供的数据结构都使用了下面的redisObject进行包装，通过包装可以提供不同场景下使用不同的数据结构的实现。Redis的对象机制还使用了**引用计数**方式的内存回收机制。

```c
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针
    void *ptr;

} robj;
```



type值可选：

```c
#define REDIS_STRING 0
#define REDIS_LIST 1
#define REDIS_SET 2
#define REDIS_ZSET 3
#define REDIS_HASH 4
```



encoding可选：

```c
#define REDIS_ENCODING_RAW 0     /* Raw representation */
#define REDIS_ENCODING_INT 1     /* Encoded as integer */
#define REDIS_ENCODING_HT 2      /* Encoded as hash table */
#define REDIS_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
#define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define REDIS_ENCODING_INTSET 6  /* Encoded as intset */
#define REDIS_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define REDIS_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
```


## 结构的编码方案

### String

编码策略如下：

- 若保存的为整数值，并且可以用long表示，使用int进行编码
- 若保存的为字符串值，并且长度大于32字节，使用SDS存储，编码格式为raw
- 若保存的为字符串值，并且长度小于32字节，使用embstr进行编码



embstr是对短字符串的优化编码，使用redisObject和sdshdr来表示字符串对象，embstr只调用一次内存分配函数来分配两个struct，raw编码使用两次内存分配函数分别完成分配。同理内存释放也如此。



### List

编码策略如下：

- 列表对象字符串对象长度小于64字节，并且保存元素数量小于512，使用ziplist进行编码，否则使用linkedlist编码
- 上述两个阈值通过`list-max-ziplist-value`和`list-max-ziplist-entries`控制





### Hash

编码策略如下：

- 保存的所有键值长度都小于64字节，并且键值对数量小于512个，满足这两个条件使用ziplist编码，否则使用hashtable编码
- 阈值由`hash-max-ziplist-value`和`hash-max-ziplist-entries`控制
- ziplist编码时，键值节点紧挨在一起，先添加的靠近表头
- hashtable编码时，键值对使用dictEntry存储，键使用`StringObject`，值使用`StringObject`



### Set

编码策略如下：

- 集合对象保存的所有元素都是整数时，并且元素个数不超过512个，使用intset进行存储，否则使用hashtable存储
- 阈值通过`set-max-intset-entries`进行控制
- intset编码时，集合存储在整数集合中
- hashtable编码时，每个键都是`StringObject`，值为NULL



### Zset



编码策略如下：

- 有序集合保存的元素数量小于128个，保存的所有成员长度都小于64字节，同时满足使用ziplist编码，否则使用skiplist编码，并使用hashtable进行辅助存储。
- 上述阈值使用`zset-max-ziplist-entries`和`zset-max-ziplist-value`来控制
- ziplist中有序集合元素按score从大到小排序。
- skiplist按照score从大到小保存了所有集合元素，支持范围查询等操作
  - 同时使用hashtable存储成员到score的映射（键为元素，值为score），通过hashtable可以在$O(1)$时间查到元素的score
  - skiplist和hashtable给你通过指针共享元素和score，因此不会浪费额外的内存。



**为什么同时使用跳表 &#43; 哈希表 来实现有序集合？**

- 若只用hashtable实现，尽管可以在$O(1)$时间内找到元素对应的score，但是无法满足范围查找，例如ZRANGE等命令，hashtable完成排序至少需要$O(N * logN)$的时间以及$O(N)$的空间。
- 若只用skiplist实现，根据元素查找score的操作升到$O(logN)$的时间复杂度。


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis%E5%AF%B9%E8%B1%A1%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/  

