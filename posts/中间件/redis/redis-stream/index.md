# Redis底层数据结构-Stream源码分析


# 简介
Redis在5.0.0版本中引进了消息队列的功能，该功能由Stream实现，本文主要介绍Stream的相关实现。


# 数据结构

## stream
Stream的实现依赖于Rax树与`listpack`结构，每个消息流都包含一个Rax树，以消息ID为key，listpack为value存储在Rax树中。其基本结构如下：

- rax：rax存储生产者生产的具体消息，每个消息有唯一ID为键
- length：代表当前stream中消息个数。
- last_id：为当前stream中最后插入的消息ID，stream为空时，该值为0。
- cgroups：存储了当前stream相关的消费组，以消费组组名为键，streamCG为值存储在rax中。
```C
typedef struct stream {
    rax *rax;               /* The radix tree holding the stream. */
    uint64_t length;        /* Number of elements inside this stream. */
    streamID last_id;       /* Zero if there are yet no items. */
    rax *cgroups;           /* Consumer groups dictionary: name -&gt; streamCG */
} stream;
```

一个Stream的基本结构如图所示：
![Stream结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/redis/stream/structure.PNG)

每一个listpack都有一个master entry，该结构存储了该listpack待插入消息的所有field。

### streamID
消息ID的基本结构如下：
- ms：消息创建时的时间
- seq：序号
```C
typedef struct streamID {
    uint64_t ms;        /* Unix time in milliseconds. */
    uint64_t seq;       /* Sequence number. */
} streamID;
```


### streamCG

每个stream会有多个消费组，每个消费组通过组名称进行唯一标识，同时关联一个streamCG结构。该结构如下：
- last_id：该消费组已经确认的最后一个消息ID
- pel：为该消费组尚未确认的消息，消息ID为键，streamNACK为值，存储在rax树中
- consumers：消费组中的所有消费者，消费者名称为键，streamConsumer为值。
```C
typedef struct streamCG {
    streamID last_id;
    rax *pel;
    rax *consumers;
} streamCG;
```

### streamConsumer
消费者结构如下：
- seen_time：为该消费者最后一次活跃的时间
- name：消费者名称，为sds结构
- pel：为该消费者尚未确认的消息，消息ID为键，streamNACK为值，存储在rax树中
```C
typedef struct streamConsumer {
    mstime_t seen_time;
    sds name;
    rax *pel;
} streamConsumer;
```

### streamNACK
该结构为未确认消息，streamNACK维护了消费组或消费者尚未确认的消息，消费组中的pel的元素与消费者的pel元素是共享的。该结构如下：
- delivery_time：为该消息最后发送给消费方的时间
- delivery_count：该消息已发送的次数
- consumer：当前归属的消费者
```C
typedef struct streamNACK {
    mstime_t delivery_time;
    uint64_t delivery_count;
    streamConsumer *consumer;
} streamNACK;
```

### streamIterator
该结构主要提供遍历功能，基本结构如下：
- stream：当前迭代器正在遍历的消息流
- master_id：为listpack中第一个插入的消息ID（master entry）
- master_fields_count：第一个entry的field个数
- master_fields_start：master entry的field首地址
- master_fields_ptr：记录field的位置
- entry_flags：当前遍历的消息的标志位
- rev：迭代器方向
- start_key,end_key：遍历范围
- ri：rax迭代器，用于遍历rax树中的所有key
- lp：当前的listpack指针
- lp_ele：当前正在遍历的listpack中的元素
- lp_flags：指向翻墙消息的flag域
- field_buf,value_buf：从listpack读取数据的缓存
```C
typedef struct streamIterator {
    stream *stream;         /* The stream we are iterating. */
    streamID master_id;     /* ID of the master entry at listpack head. */
    uint64_t master_fields_count;       /* Master entries # of fields. */
    unsigned char *master_fields_start; /* Master entries start in listpack. */
    unsigned char *master_fields_ptr;   /* Master field to emit next. */
    int entry_flags;                    /* Flags of entry we are emitting. */
    int rev;                /* True if iterating end to start (reverse). */
    uint64_t start_key[2];  /* Start key as 128 bit big endian. */
    uint64_t end_key[2];    /* End key as 128 bit big endian. */
    raxIterator ri;         /* Rax iterator. */
    unsigned char *lp;      /* Current listpack. */
    unsigned char *lp_ele;  /* Current listpack cursor. */
    unsigned char *lp_flags; /* Current entry flags pointer. */
    /* Buffers used to hold the string of lpGet() when the element is
     * integer encoded, so that there is no string representation of the
     * element inside the listpack itself. */
    unsigned char field_buf[LP_INTBUF_SIZE];
    unsigned char value_buf[LP_INTBUF_SIZE];
} streamIterator;
```

## listpack
listpack是一个字符串列表的序列化格式，该结构可用于存储字符串或整型。其主要结构如下图所示：
![listpack基本结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/redis/stream/listpack.PNG)
listpack主要由四部分构成，分别是：
- Total Bytes为整个listpack的空间大小
- Num Elem：listpack的Entry个数，占用两个字节，但是Entry个数大于等于65535时，该值为65535，所以这种情况下获取元素个数，需要遍历整个listpack
- Entry：为每个具体的元素
- End：为listpack的结束标志，占用一个字节，内容为0xFF

Entry由三部分构成，基本如下：
- Encode：元素的编码方式，占用一个字节
- content：内容
- backlen：记录entry的长度（不包括该字段本身）

其中编码方式取值如下图所示：
![编码取值](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/redis/stream/encode.PNG)

Stream的消息内容存储在listpack中，消息存储格式是每个字段都是一个entry，而不是键整个消息作为字符串存储的，每个listpack会存储多个消息，具体存储个数由`stream-node-max-bytes`（listpack节点最大占用字节数，默认4096）和`stream-node-max-entries`（最大存储元素个数，默认100）决定.

每个listpack在创建时，会将第一个插入的entry构建成master entry，其基本结构如下所示：
```
count   |   deleted  |  num-fields  |   field-1 |   field-2 |   0
```
其中：
- count：为当前listpack中所有未删除的消息个数
- deleted：当前listpack中所有已经删除的消息个数
- num-fields：field个数
- field-N：field域
- 0为标识位，再从后向前遍历listpack时使用

存储一个消息时，如果该消息的field域与master entry完全相同，则不需要再次存储field域


## Rax
Redis对于Rax的解释为`A radix tree implement`，基数树的一种实现。Rax中不仅可以存储字符串，也可以为该字符串设置值形成kv结构。其基本结构如下：
- head：指向头节点
- numele：元素个数（key）
- numnodes：节点个数
```C
typedef struct rax {
    raxNode *head;
    uint64_t numele;
    uint64_t numnodes;
} rax;
```


## raxNode
raxNode代表Rax树中的一个节点，其基本结构如下所示：
- iskey：表明当前节点是否包含一个key，占用1bit
- isnull：表明当前key对应的value是否为空，占用1bit
- iscompr：表明当前节点是否为压缩节点，占用1bit
- size：压缩节点压缩的字符串长度或者非压缩节点的子节点个数，占用29bit
- data：包含填充字段，同时存储了当前节点包含的字符串以及子节点的指针，key对应的value指针。
```C
typedef struct raxNode {
    uint32_t iskey:1;
    uint32_t isnull:1;
    uint32_t iscompr:1;
    uint32_t size:29;
    unsigned char data[];
} raxNode;
```

其中raxNode分为压缩节点域非压缩节点。主要区别在于非压缩节点的每个字符都有子节点，如果字符个数小于2，都是非压缩节点。

## raxStack
raxStack结构用于存储从根节点到当前节点的路径，基本结构如下：
- stack：用于记录路径，该指针可能指向static_items或者堆内存
- items,maxitems：代表stack指向的空间的已用空间以及最大空间
- static_items：一个数组，每个元素都是指针，存储路径
- oom：代表当前栈是否出现过内存溢出
```C
typedef struct raxStack {
    void **stack;
    size_t items, maxitems;
    void *static_items[RAX_STACK_STATIC_ITEMS];
    int oom;
} raxStack;
```

## raxIterator
raxIterator用于遍历Rax树中的所有key，基本结构如下：
- flags：代表当前迭代器标志位,取值如下：
    - RAX_ITER_JUST_SEEKED：当前迭代器指向的元素是刚刚搜索过的，当需要从迭代器中获取元素时，直接返回当前元素并清空标志位。
    - RAX_ITER_EOF：代表当前迭代器已经遍历到最后一个节点
    - RAX_ITER_SAFE：代表当前迭代器为安全迭代器，可以进行写操作
- rt：当前迭代器对应的rax
- key：存储了当前迭代器遍历到的key，该指针指向key_static_string或者堆内存
- data：value值
- key_len,key_max：key指向的空间的已用空间以及最大空间
- key_static_string：key的默认存储空间，key过大时，会使用堆内存
- node：当前key所在的raxNode
- stack：记录了从根节点到当前节点的路径，用于raxNode线上遍历
- node_cb：为节点的回调函数，默认为空
```C
typedef struct raxIterator {
    int flags;
    rax *rt;                /* Radix tree we are iterating. */
    unsigned char *key;     /* The current string. */
    void *data;             /* Data associated to this key. */
    size_t key_len;         /* Current key length. */
    size_t key_max;         /* Max key len the current key buffer can hold. */
    unsigned char key_static_string[RAX_ITER_STATIC_LEN];
    raxNode *node;          /* Current node. Only for unsafe iteration. */
    raxStack stack;         /* Stack used for unsafe iteration. */
    raxNodeCallback node_cb; /* Optional node callback. Normally set to NULL. */
} raxIterator;
```

# 总结
上文主要对Redis Stream的基本结构与其底层数据结构做了简要分析，了解了消息由listpack结构存储，以消息ID为key，listpack为value存储在Rax树中。

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis-stream/  

