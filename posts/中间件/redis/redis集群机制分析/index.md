# Redis集群机制分析


## 主从复制模式

Redis集群实现数据同步，保证服务高可用。

- 一主多从

- 数据流向是单向的，master-&gt;slave



开启指令：

```
slaveof 192.168.1.10 6379		//当前服务节点称为指定IP的从节点
slaveof no one                  //取消从属关系
```



配置文件：

```
slaveof ip port
slave-read-only yes             //从节点只读
```



全量复制的开销：
- bgsave的时间
- RDB文件网络传输时间
- 从节点清空数据时间
- 从节点加载RDB的时间
- 可能的AOF重写时间



### 实现

Redis复制分为同步和命令传播两个操作：

- 同步：将从服务器的数据库状态更新为主库状态
- 命令传播：同步完后，主库状态又发生变化，此时使用命令传播完成状态同步。



Redis使用`PSYNC`可以同时解决全量复制和部分复制两种情况。

需要维护的变量包括 ：

- runID：每个Redis实例启动生成的随机ID
- offset：复制进度



执行`slaveof`时，slave节点会将master的地址保存在：

```c
struct redisServer {
    char *masterhost;
    int masterport;
}
```

slave节点会将该命令封装成通信协议并发给master，完成socket建立，并发送PING检查socket是否正常。



### 心跳检测

命令传播的阶段，slave默认每秒一次的频率，向master发送心跳：

```
REPLCONF ACK &lt;replication_offset&gt;

replication_offset为slave当前的复制偏移量
```



心跳主要有三个作用：

- 检测主从之间连接状态：节点保活机制
- 辅助实现min-slaves：防止master在不安全的情况下执行写命令。
  - `min-slaves-to-write`
  - `min-slaves-max-log`
- 检测命令丢失：如果因为网络原因产生命令丢失，master发现slave的offset小于自己当前offset，则认为产生命令丢失，并按照slave的offset传递数据。







## sentinel机制



主从复制存在的问题

- 手动故障转移：master发生宕机，需要手动切换
- 写能力和存储能力受限



Sentinel（机制）是Redis官方推荐的高可用性(HA)解决方案，由一个或多个sentinel组成的sentinel system，可以监视主从集群的状态，当master发生宕机，自动将master下面的某个slave升级为新的master。



### 启动

```
./redis-sentinel /path/to/sentinel.conf
```

启动流程会执行以下步骤：

- 初始化server：sentinel本质上是一个运行在特殊模式的Redis服务器。
- 代码：sentinel使用`sentinel.c/REDIS_SENTINEL_PORT`作为默认端口，使用`sentinel.c/sentinelcmds`作为命令表
- 初始化sentinel状态：`sentinel.c/sentinelState`结构
- 根据配置文件，初始化sentinel监视的主服务器列表：上面的结构使用`dict`保存master信息，key为master的名称，value为master对应的`sentinel.c/sentinelRedisInstance`，每个被监视的Redis服务器实例都被使用该结构存储。master的IP端口信息使用`struct sentinelAddr`进行存储。该数据sentinel初始化时从`sentinel.conf`完成加载。
- 创建连向master的socket连接：sentinel节点对master创建两个异步连接：
  - 命令连接：专门向master发送命令，并接收回复
  - 订阅连接：订阅master的`__sentinel__:hello`频道，用于sentinel发现其他sentinel节点



### 获取信息

每10s每个sentinel会对master发送`info`命令，sentinel主要获取两方面信息：

- master信息：run ID、role
- master对应的slave信息，包括IP端口、状态，偏移量等



当获取到对应slave节点信息后，sentinel会创建到slave的两个连接，创建完命令连接后， 每10s对slave发送`info`命令，sentinel会获取以下信息：

- slave：run ID、role、master_port、master_host、主从之间的连接状态



sentinel默认每2s向所有节点的`__sentinel__:hello`发送消息，主要是sentinel本身信息和master的信息。并且sentinel会通过订阅连接不断接收主从节点发来的消息，这里面是sentinel信息，如果与当前sentinel不同，说明是其他sentinel，那么当前sentinel会更新sentinel字典信息，并创建连向新sentinel的命令连接。



### 主观下线

sentinel每1s向所有实例发送PING，包括主从、其他sentinel，并通过实例是否返回PONG判断是否在线。配置文件中的`down-after-milliseconds`内，未收到回复，sentinel就会将该实例标记为主观下线状态。但是多个sentinel的主线下线阈值不同，可能存在某节点在sentinelA是离线，在sentinelB是在线。



### 客观下线

多个sentinel实例对同一台服务器做出SDOWN判断，并且通过`sentinel is-master-down-by-addr ip`命令互相交流之后，得出服务器下线的判断。客观下线是足够的sentinel都将服务器标记为主观下线之后，服务器才会标记为客观下线。



### sentinel选举

当某个master被判断为客观下线后，各个sentinel会进行协商，选举出一个主sentinel，并由主sentinel完成故障转移等操作。选举算法是Raft，**选举规则**如下：

- 所有sentinel拥有相同资格参与选举、每次完成选举后，epoch都会自增一次
- 当A向B发送`sentinel is-master-down-by-addr ip`时，A要求B将其设置为主sentinel。最先向B发送的将会成为B的局部主sentinel。
- B接收到A的命令，回复信息中分别由局部主sentinel的run ID和`leader_epoch`。
- A接收到回复，B的局部主sentinel是否为自己，如果A称为半数以上的局部主sentinel，则会成为全局主sentinel。
- 规定时间内未完成选举，则会再进行。





### 故障转移

选举出主sentinel后，主sentinel会将已下线的master进行故障转移，步骤如下：

- 已下线的master从属的slave列表中挑选一个，并将其转换为master。挑选其中状态良好、数据完整的slave。
- 让其他slave改为复制新的master的数据。向其他slave发送`SLAVEOF`。





## Redis Cluster

Redis集群是Redis提供的分布式数据库方案，通过sharding完成数据共享，并通过复制、故障转移来提供高可用特性。



### Node

一个集群由多个Node构成，刚运行时都是独立的Node，需要通过`CLUSTER MEET &lt;ip&gt; &lt;port&gt;`指令来构成集群。`CLUSTER NODE`可以查询集群信息。Node通过以下配置参数判断是否以集群模式启动：

```
cluster-enabled yes
```

Node使用以下struct来定义：

```c
// 节点状态
struct clusterNode {

    // 创建节点的时间
    mstime_t ctime; /* Node object creation time. */

    // 节点的名字，由 40 个十六进制字符组成
    // 例如 68eef66df23420a5862208ef5b1a7005b806f2ff
    char name[REDIS_CLUSTER_NAMELEN]; /* Node name, hex string, sha1-size */

    // 节点标识
    // 使用各种不同的标识值记录节点的角色（比如主节点或者从节点），
    // 以及节点目前所处的状态（比如在线或者下线）。
    int flags;      /* REDIS_NODE_... */

    // 节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch; /* Last configEpoch observed for this node */
    
    // 该节点负责处理的槽数量
    int numslots;   /* Number of slots handled by this node */

    // 如果本节点是主节点，那么用这个属性记录从节点的数量
    int numslaves;  /* Number of slave nodes, if this is a master */

    // 指针数组，指向各个从节点
    struct clusterNode **slaves; /* pointers to slave nodes */

    // 如果这是一个从节点，那么指向主节点
    struct clusterNode *slaveof; /* pointer to the master node */

    // 最后一次发送 PING 命令的时间
    mstime_t ping_sent;      /* Unix time we sent latest ping */

    // 最后一次接收 PONG 回复的时间戳
    mstime_t pong_received;  /* Unix time we received the pong */

    // 最后一次被设置为 FAIL 状态的时间
    mstime_t fail_time;      /* Unix time when FAIL flag was set */

    // 最后一次给某个从节点投票的时间
    mstime_t voted_time;     /* Last time we voted for a slave of this master */

    // 最后一次从这个节点接收到复制偏移量的时间
    mstime_t repl_offset_time;  /* Unix time we received offset for this node */

    // 这个节点的复制偏移量
    long long repl_offset;      /* Last known repl offset for this node. */

    // 节点的 IP 地址
    char ip[REDIS_IP_STR_LEN];  /* Latest known IP address of this node */

    // 节点的端口号
    int port;                   /* Latest known port of this node */

    // 保存连接节点所需的有关信息
    clusterLink *link;          /* TCP/IP link with this node */

    // 一个链表，记录了所有其他节点对该节点的下线报告
    list *fail_reports;         /* List of nodes signaling this as failing */

};

typedef struct clusterLink {

    // 连接的创建时间
    mstime_t ctime;             /* Link creation time */

    // TCP 套接字描述符
    int fd;                     /* TCP socket file descriptor */

    // 输出缓冲区，保存着等待发送给其他节点的消息（message）。
    sds sndbuf;                 /* Packet send buffer */

    // 输入缓冲区，保存着从其他节点接收到的消息。
    sds rcvbuf;                 /* Packet reception buffer */

    // 与这个连接相关联的节点，如果没有的话就为 NULL
    struct clusterNode *node;   /* Node related to this link if any, or NULL */

} clusterLink;
```





### 分片方式

集群通过**分片**的方式保存数据，当请求进入，计算key的hash值，并对16383进行取余，得到指定slot位置 。整个数据库被分为16384个slot，这些slot会被平均分配给各个节点。slot被定义在`clusterNode`中，整个slot数组长度为2048 bytes，以`bitmap`形式进行存储。处理slot的node的映射信息存储在`clusterState`中。

```c
// 槽数量
#define REDIS_CLUSTER_SLOTS 16384

// 由这个节点负责处理的槽
// 一共有 REDIS_CLUSTER_SLOTS / 8 个字节长
// 每个字节的每个位记录了一个槽的保存状态
// 位的值为 1 表示槽正由本节点处理，值为 0 则表示槽并非本节点处理
// 比如 slots[0] 的第一个位保存了槽 0 的保存情况
// slots[0] 的第二个位保存了槽 1 的保存情况，以此类推
unsigned char slots[REDIS_CLUSTER_SLOTS/8];

// 该节点负责处理的槽数量
int numslots;


// 存储处理各个槽的节点
// 例如 slots[i] = clusterNode_A 表示槽 i 由节点 A 处理
clusterNode *slots[REDIS_CLUSTER_SLOTS];
```

至于为什么需要存储slot-&gt;node的映射，是为了解决**想知道slot[i]由哪个node存储**，使用该映射能在$O(1)$内查询到，否则需要遍历`clusterState.nodes`字典中的所有`clusterNode`结构，检查这些结构的slots数组，直到找到负责处理slot{i]的节点为止。



```
# 将slot分配给指定节点
CLUSTER ADDSLOTS ＜slot＞ [slot ...]
```



### 命令处理

假设客户端向集群发送key相关指令，处理流程如下：

1. 接受命令的Node计算出key属于哪个slot，并判断当前slot是否属于自己，是则执行
   - 通过`CRC16(key) &amp; 16383`计算key的所属slot
2. 否则给客户端返回MOVED错误（附带处理该slot的Node信息），客户端重新给正确Node发送请求。
   - `MOVED ＜slot＞ ＜ip＞:＜port＞`



### Gossip协议

Cluster底层通信协议为Gossip协议，集群中每个节点定期向其他节点发送PING，以此交换各个节点状态。



### 故障转移

集群中节点定期向其他节点发送PING，检测对方是否在线，如果未收到PONG，则会将其标记为`probable fail`，如果集群中半数以上认为某节点疑似下线。则会向其他节点广播某节点`fail`的情况。

当某个slave发现自己正在复制的master进入fail状态，则会进行故障转移：

- 选取master中一个slave，并执行`SLAVEOF no one`，成为新的master
- 新的master会撤销所有已下线的master的指派slot，并将这些都指派给自己
- 新的master广播一条PONG消息，通知其他节点其已成为master



### 集群选主

Redis Cluster的选举算法基于Raft实现。

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis%E9%9B%86%E7%BE%A4%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/  

