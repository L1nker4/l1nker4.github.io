# Kafka学习笔记(四)-集群工作机制




## Controller机制

Controller主要作用是在Zookeeper的帮助下管理和协调整个Kafka集群（在zk中存储集群元数据）。Kafka集群中会有一个或多个Broker，其中一个Broker会被选举为Controller，它负责管理整个集群中所有分区和副本的状态，其工作职责包括以下内容：

- **Topic管理**：完成对Kafka的Topic的创建删除、分区增加等操作。
- **分区重分配**：新的Broker加入集群时，不会自动分担已有的topic负载，只会对后续的topic生效，此时如果需要对已有topic负载，需要用户手动进行**分区重分配**。
- **Leader选举**：负责Partition Leader选举的工作
- 集群成员管理：
  - Kafka 使用Zookeeper的临时节点来选举Controller
  - Zookeeper在Broker加入集群或退出集群时通知Controller
  - Controller负责在Broker加入或离开集群时进行分区Leader选举
- 元数据管理：Controller负责管理集群中所有的元数据



Controller选举流程：

- 每个Broker启动时，都会尝试读取`/controller`节点的brokerid的值，如果值不为-1，则表明已经有其他broker节点成为Controller，当前broker放弃选举
- 如果不存在`/controller`节点或节点数据异常，则主动创建节点并存储brokerid
- 其他broker会将选举成功的Brokerid都在内存保存下来
- 同时使用`/controller_epoch`持久性节点来记录任期号，记录Controller发生变化的次数，类似于Raft中的任期。
  - 初始值为1，每个与Controller交互的请求都会携带`controller_epoch`，如果请求的`controller_epoch`大于内存中`controller_epoch`，说明内存中的值过期了，目前已有新的Controller当选。
  - 由两部分组成：
    - epoch：单调递增的版本号，leader发生变更，进行自增
    - start offset：Leader副本在该Epoch值上写入的首条消息的位移。
  - 每个分区都缓存该值，并定期持久化到`checkpoint`文件中



### Partition Leader选举

Controller拥有选举分区Leader的功能，每个分区都会有一个Broker作为Leader，处理所有的读写请求，选举流程由Controller负责：

1. Controller从ZK中读取当前分区所有的ISR集合
2. 调用配置的分区选择算法选举分区的Leader

`Partition Leader`的定义如下：

&gt; Each partition has one server which acts as the &#34;leader&#34; and zero or more servers which act as &#34;followers&#34;. The leader handles all read and write requests for the partition while the followers passively replicate the leader. If the leader fails, one of the followers will automatically become the new leader. Each server acts as a leader for some of its partitions and a follower for others so load is well balanced within the cluster.



触发Partition Leader选举的时机如下：

- Offline：创建新分区或现有分区Leader离线
- Reassign：用户执行重分配操作
- PreferredReplica：leader迁移回首选副本
- 分区的现有leader即将下线



### 故障转移

如果Controller发生故障（宕机或网络超时等），Kafka能够立即启用备用Controller来代替之前的，这个过程称为`Failover`。其基本流程如下：

1. ZK检测到当前controller宕机（ZK的watch机制），删除`/controller`临时节点。
2. 某个Broker抢先注册了`/controller`临时节点，成为新的controller
3. 该Broker从ZK拉取元数据，并执行初始化。



## 分区副本机制



副本对分区数据进行冗余存储，以保证系统的高可用和数据一致性，当分区Leader发生宕机或数据丢失，可以从其他副本获取分区数据。分区Leader负责处理分区所有的读写请求。



每个分区都有一个ISR集合，用于维护所有同步并可用的副本，ISR最少拥有Leader副本这一个元素，对于Follower来说，必须满足以下条件才是ISR：

- 必须定时向Zookeeper发送心跳
- 在规定时间内从Leader获取过消息

若副本不满足以上两条件，就会从ISR列表移除。



`replica.lag.time.max.ms`：Follower能落后Leader的最长时间间隔，默认为10s



使用Zookeeper来管理ISR，节点位置为`/brokers/topics/[topic]/partitions/[partition]/state`



### HW和LEO机制



几个offset概念：

- base offset：副本中第一条消息的offset
- HW：high watermark，副本中最新一条已提交消息的offset
  - 用来标识分区中哪些消息可以消费，HW之后的数据对消费者不可见
  - 完成副本数据同步的任务
  - HW永远小于LEO
- LEO：log end offset，副本中下一条待写入消息的offset
  - 用于更新HW：follower和Leader的LEO同步后，两者HW就可以更新



Leader除了保存自己的一组HW和LEO，也会保存其他Follower的值，目的是帮助Leader副本确定分区高水位。



### 更新机制

- Follower从Leader同步数据时，会携带自己的LEO值，Leader将Follower中最小的LEO作为HW值，这是为了保证Leader宕机后，新选举的节点拥有所有HW之前的数据，保证了数据安全性。
- Follower获取数据时，请求Leader的HW，然后follower.HW = min(follower.LEO, leader.HW)


## Consumer Group


- 每个Group有一个或多个Consumer
- 每个Group可以管理0、1、多个Topic
- 每个Group拥有一个全局唯一的id
- Group下的每个Consumer Client可以同时订阅Topic的一个或多个Partition
- Group下同一个Partition只能被一个Consumer订阅


#### Group分配策略


一个Group中有多个Consumer，一个Topic有多个Partition，Group存在一个分配的情况：**确定某个Partition由哪个Consumer来消费**。


Kafka提供了三种分区分配策略，也支持用户自定义分配策略：

- **RangeAssignor**：默认分区分配算法，按照Topic的维度进行分配，Partition按照分区ID进行排序，然后对订阅此Topic的Group内Consumer及逆行排序，之后均衡的按照范围区段将分区分配给Consumer。
  - 缺点：当Consumer增加，不均衡的问题就会越来越严重。
- **RoundRobinAssignor**：将Group内订阅的Topic所有Partition以及所有Consumer进行排序后按照顺序尽量均衡的一个个分配
- **StickyAssignor**：发生rebalance时，分配结果尽量与上次分配保持一致。



#### 心跳保活机制

Consumer加入到Group中之后，维持状态通过两种方式实现：
1. consumer会判断两次poll的时间间隔是否超出max.poll.interval.ms设定的值，超过则主动退出Group。
2. 异步线程定时发送心跳包，间隔超出session.timeout.ms则broker判定consumer离线，主动将其退出Group。


若consumer被移出Group，会触发Rebalance。


### Coordinator

Coordinator用于存储Group的相关Meta信息，主要负责：
1. Group Rebalance
2. offset管理：将对应Partition的Offset信息记录到__consumer_offsets中
3. Group Consumer管理

Broker启动时，都会创建对应的Coordinator组件，每一个Consumer Group，Kafka集群都会给它选一个broker作为其Coordinator



### Rebalance


基于给定的分配策略，为Group下所有的Consumer重新分配所订阅的Partition。

目的：重新均衡消费者，尽量达到最公平的分配



触发时机：
1. 组成员数量发生变化
	- 新加入consumer
	- 已有consumer退出（挂掉、调用unsubscribe）
1. 订阅topic数量发生变化
2. 订阅topic的分区数发生变化

影响：
- 可能重复消费，consumer还未提交offset，被rebalance
- Rebalance期间，整个Group内的Consumer无法消费消息
- 集群不稳定：只要做变动，就发生一次rebalance
- 影响消费速度：频繁的rebalance

如何避免频繁rebalance？
1. session.timeout.ms：超时时间，默认10s
2. heartbeat.interval.ms：心跳间隔，默认3s
3. max.poll.interval.ms：每隔多长时间去拉取消息，超过阈值则被coordinator


## 消息投递的三种语义


- At most once：消息发送或消费至多一次
	- Producer：发送完消息，不会确认消息是否发送成功，存在丢失消息的可能性。
	- Consumer：对一条消息最多消费一次，可能存在消费失败依旧提交offset的情况 。
- at least once：消息发送或消费至少一次
	- Producer：发送完消息会确认是否发送成功，如果没有收到ACK，会不断重复发送
	- Consumer：对一条消息可能消费多次。
- exactly once：消息恰好只发送或消费一次
	- Producer：消息的发送是幂等的，broker不会有重复的消息。
	- Consumer：消费和提交offset的动作是原子性的。




## 幂等性Producer

现象：producer在进行重试的时候可能会重复写入消息，使用幂等性可以避免这种情况。

配置项：enable.idempotence=true

每个producer初始化时，会设置唯一的ProducerID，broker发送消息时会带上ProducerID并为每个消息分配一个seq number，broker端根据pid和seq number去重

特点：
1. 只能保证单分区上的幂等性
2. 只能实现单会话的幂等性，producer重启后，幂等失效。


## 事务

Kafka中事务特性用于以下两种场景：

1. producer生产消息和consumer消费消息并提交offset在一个事务中，要么同时成功，要么同时失败。
2. producer将多条消息封装在一个事务中，要么同时发送成功、要么同时发送失败。






## Ops问题


### PageCache竞争

现象：同一台broker上的不同partition间竞争PageCache资源，相互影响，导致整个Broker处理延迟上升。

- Producer：Server端的IO线程统一将请求的消息数据写入到PageCache之后立即返回，当达到指定大小后，会刷回到硬盘中。
- Consumer：利用OS的ZeroCopy机制，Broker接收到读请求时，向OS发生sendfile系统调用，OS接收后尝试从PageCache中获取数据，如果数据不存在，触发缺页中断从硬盘中读取数据到buffer，随后DMA操作直接将数据拷贝到网卡Buffer中等待后续TCP传输。




## 常见问题



### Kafka如何保证消息有序

Kafka中并不保证消息全局有序，只能保证**分区有序**。

- 生产者：使用同步方式发送，设置`acks = all &amp;  max.in.flight.requests.per.connection = 1`，存在下面的问题：
  - 重发问题：发送失败，需要判断是否自动重试，设置`retries &gt; 0`
  - 幂等问题：设置`enable.idempotence = true`，设置后生产者客户端会给消息添加序列号，每次发送把序列号递增1
- 服务端Broder：Kafka只能保证单个Partition内消息有序，若要Topic全局有序，需要设置单分区。
- 消费者：根据Group机制，Topic下的分区只能从属group下的某一个消费者，若Consumer单线程消费没有问题，多线程并发消费顺序会乱。



### 如何处理Kafka大量消息积压

出现消息积压的原因：Kafka集群中出现了**性能问题**，来不及消费消息。



解决：

- 生产端性能优化

  - 检查发送消息的业务逻辑，合理设置并发数量和批量大小，注意消息体大小。
  - 可以动态配置开关，关闭MQ生产，并在解决积压问题后，通过**消息补偿机制**进行补偿，**消费端需要支持幂等**。

- 消费端性能优化

  - 自动提交改为手动提交
  - 单条消息消费改为批量消费
  - 涉及DB操作，一定要检查是否存在慢SQL
  - 一定要保证消费端速度大于生产端的生产速度，扩容消费端实例来提升总体的消费能力。要注意在扩大Consumer实例的同时，必须扩容Topic的Partition数量，确保两者数量大致相等。

- 查看Broker端日志、监控系统等情况，是否存在硬件层面的问题：磁盘空间等。

  - partition分区数 &gt; 应用消费节点时，可以扩大Broker分区数量，否则扩broker分区没用。


### Kafka性能瓶颈

常见问题：

1. Kafka单机topic超过64个分区
	- load负载加大
	- 消息延迟加大
2. 一个分区一个文件
3. 单个文件按照append写是顺序IO
4. 分区多文件多，局部的顺序IO会退化到随机IO


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%9B-%E9%9B%86%E7%BE%A4%E5%B7%A5%E4%BD%9C%E6%9C%BA%E5%88%B6/  

