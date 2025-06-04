# Kafka学习笔记(一)-基础入门


## 简介

Kafka是由LinkedIn使用Scala语言开发的分布式消息引擎系统，目前已被捐献给Apache基金会，它以高吞吐量、可持久化、流数据处理等特性而被广泛使用。它主要有以下三种主要功能：

- 消息中间件：具备常见的消息队列功能：系统解耦、冗余存储、流量削峰填谷、缓冲、异步通信，同时具备消息顺序性保障、回溯消费等功能。
- 数据存储系统：使用Kafka存储各种服务的log，然后统一输出，ELK可使用Kafka进行数据中转。
- 流数据处理平台：与flink、spark、storm等组件整合，提供实时计算。



Kafka支持两种常见消息传输模型：

- **点对点模型**：也称为消息队列模型，系统A发送的消息只能被系统B接收，其它系统读取不到。
- **发布/订阅模型**：使用`Topic`接收消息，`Publisher`和`Subscriber`都可以有多个，可以同时向Topic发送接收消息。



## 基本概念

- Kafka体系结构：一个Kafka集群包括若干Producer、Customer、Broker，以及一个Zookeeper集群。
  - Broker：服务端由被称为`Broker`的服务进程构成，`Broker`负责接受和处理客户端请求，以及对消息进行持久化。
    - 可以简单看作为一个独立的Kafka服务节点（进程示例）
    - Broker层面的领导者被称为Controller
  - Producer：客户端节点，发送消息的一方。
  - Customer：客户端节点，接收消息的一方。
  - Customer Group：消费者组内每个消费者负责消费不同分区的数据。一个分区只能由组内一个消费者消费，不同消费组之间互不影响。
  - &lt;del&gt;Zookeeper集群：负责元数据管理，集群选举。&lt;/del&gt;目前最新版3.1.0提供了KRaft模式，集群不再依赖ZK。
    - ZK主要负责存储Kafka集群的元数据，协调集群工作。
    - 记录信息如下：
	    - 1. /brokers/ids/{0-n}：记录broker服务器节点，不同的broker使用不同的brokerid，会将自己的ip地址和端口信息记录到节点
	    - 2. /brokers/topics/{topic}：记录topic分区以及broker的对应信息
	    - 3. /comsumers/{group_id}/ids/{consumer_id}：消费者负载均衡
- Topic：**逻辑概念**，Kafka中消息以topic为单位进行分类，生产者将消息发送到特定的topic，消费者订阅topic进行消费。


### 分区（partition）
- Partition：topic可以分为多个partition，分区在物理存储层面可以看作一个可Append的Log文件，消息被Append到Log中会分配一个`offset`，这个属性是消息的唯一标识 ，**Kafka通过它来保证消息在分区内的顺序性**，因此Kafka**保证分区有序**而不是主题有序。
  - 主题中的partition可以分布在不同的Broker中。
  - 消息到达broker后，根据分区规则存储到指定的partition。



### 多副本机制（Replica）

- 多副本机制（Replica）：是对于**分区**而言的，**同一分区的不同副本中保存的是相同的消息。**
  - Leader：分区中的主副本，负责处理读写请求，Producer/Consumer交互的对象。
  - Follower：分区中的从副本，只会实时从Leader副本同步数据。
  - 所有副本被称为AR（Assigned Replicas），所有与Leader副本数据一致性差距过多的副本组成OSR（Out-of-Sync Replicas），于leader保持一定程度同步的副本称为ISR（In-Sync Replicas）。
  - Leader故障后，从ISR中选举新的Leader。
  - 高水位（HW-High Watermark）：消费者能消费的最大offset位置，相当于**所有副本中都存在的消息**（木桶效应）
  - LEO（Log End Offset）：标识当前日志文件中下一条待写入消息的offset，每个副本都会维护自身的LEO，**ISR中最小的LEO即为分区的HW**。


多副本的作用：提高Kafka的可用性。

涉及参数：
- unclean.leader.election.enable：为true则ISR为空也能选举，为false则只能从ISR选举。




### metadata

Q：客户端如何知道请求哪个broker？

client通过metadata从任意broker获取集群信息，其中包括：
1. topic信息
2. 每个topic的分区、副本情况
3. leader分区所在的broker连接信息
4. 每个broker的连接信息
5. 其他信息

![Kafka应用架构(https://developer.confluent.io/courses/architecture/get-started/)](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20230718224608.png)



## 部署

使用WSL 2环境进行单机部署。

```
zero@Pluto:~$ uname -a
Linux Pluto 4.19.128-microsoft-standard #1 SMP Tue Jun 23 12:58:10 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```



Kafka需要Java环境，由于Kafka最新版本3.1.0不再支持Java8，故使用Java11。

```
zero@Pluto:~$ java -version
openjdk version &#34;11.0.15&#34; 2022-04-19
OpenJDK Runtime Environment (build 11.0.15&#43;10-Ubuntu-0ubuntu0.18.04.1)
OpenJDK 64-Bit Server VM (build 11.0.15&#43;10-Ubuntu-0ubuntu0.18.04.1, mixed mode, sharing)
```



```
cd /opt
wget https://dlcdn.apache.org/kafka/3.1.0/kafka_2.13-3.1.0.tgz
tar -zxvf kafka_2.13-3.1.0.tgz
cd kafka_2.13-3.1.0
```



运行下面命令单机部署：

```
bin/kafka-server-start.sh config/server.properties
```



### 创建主题测试

创建一个topic名称为test，副本因子为1，分区个数为1的Topic。

```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```



通过`--describe`可以进行查看。

```
bin/kafka-topics.sh --describe --topic test --bootstrap-server localhost:9092
Topic: test     TopicId: Zb5Tr1MpS1ukdX22mnoBoQ PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: test     Partition: 0    Leader: 0       Replicas: 0     Isr: 0
```





### 发送/消费消息测试

`bin/kafka-console-producer.sh`可以通过命令行输入消息并发送给Kafka，每一行是一条消息。

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

同时提供了`bin/kafka-console-consumer.sh`提供消费消息的功能。

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```





### 配置参数

```
# 指明连接的Zookeeper集群服务地址，使用逗号进行分割
# 
zookeeper.connect=localhost:2181

# 指定kakfa集群中broker的唯一标识
broker.id=0

# Kafka日志文件位置
log.dirs=/opt/kafka_2.13-2.8.1/logs

#  定义Kafka Broker的Listener
listeners=PLAINTEXT://:9092

#将Broker的Listener信息发布到Zookeeper中
advertised.listeners=PLAINTEXT://172.19.143.59:9092
```



### 部署kafka-map

`kafka-map`是使用`Java11`和`React`开发的一款`kafka`可视化工具。

```
docker run -d \
    -p 8080:8080 \
    -v /opt/kafka-map/data:/usr/local/kafka-map/data \
    -e DEFAULT_USERNAME=admin \
    -e DEFAULT_PASSWORD=zero... \
    --name kafka-map \
    --restart always dushixiang/kafka-map:latest
```





## 应用场景

- 消息队列：可以替代传统消息队列，比如ActiveMQ、RabbitMQ等。
- 流处理平台：对数据进行实时流处理。
- 网站活动追踪：用户活动（浏览网页、搜索等）、网站活动发布到不同的主题，进行实时处理监测，可替代Hadoop或其他离线数仓。
- 日志聚合
- 事件采集





## 使用

Kafka有5个核心API：

- Producer API：客户端发送消息到Kafka集群中的Topic。
- Consumer API：客户端从Kafka集群读取消息。
- Streams API：允许从输入topic转换数据流到输出topic。
- Connect API：通过实现`connector`，不断从源系统拉取数据到Kafka，或者从Kafka提交数据到系统。
- Admin API：用于检查和管理topic、broker等资源。





Maven工程添加如下依赖：

```xml
&lt;dependency&gt;
    &lt;groupId&gt;org.apache.kafka&lt;/groupId&gt;
    &lt;artifactId&gt;kafka-clients&lt;/artifactId&gt;
    &lt;version&gt;3.1.0&lt;/version&gt;
&lt;/dependency&gt;
```





### 生产者 API

生产者的缓冲池保存尚未发送给服务端的消息，后台的IO线程负责将消息转换为请求发送给服务端，**如果使用后不关闭生产者，会丢失这些消息。**



```java
public static void main(String[] args) {
        Properties props = new Properties();
        props.put(&#34;bootstrap.servers&#34;, &#34;172.28.203.172:9092&#34;);
        props.put(&#34;acks&#34;, &#34;all&#34;);
        props.put(&#34;retries&#34;, 0);
        props.put(&#34;batch.size&#34;, 16384);
        props.put(&#34;linger.ms&#34;, 1);
        props.put(&#34;buffer.memory&#34;, 33554432);
        props.put(&#34;key.serializer&#34;, &#34;org.apache.kafka.common.serialization.StringSerializer&#34;);
        props.put(&#34;value.serializer&#34;, &#34;org.apache.kafka.common.serialization.StringSerializer&#34;);

        Producer&lt;String, String&gt; producer = new KafkaProducer&lt;&gt;(props);
        for(int i = 0; i &lt; 100; i&#43;&#43;) {
            producer.send(new ProducerRecord&lt;String, String&gt;(&#34;test&#34;, Integer.toString(i), Integer.toString(i)));
        }

        producer.close();
    }
```



#### 配置参数

`Properties`各配置字段含义：

- bootstrap.servers：Kafka集群的地址，集群地址使用逗号隔开。
- acks：指定分区中必须要有多少个副本收到这条消息，之后生产者认为这条消息是成功写入的。取值如下：
  - 1：只要Leader副本成功写入，就会收到来自服务端的成功响应。
  - 0：生产者发送消息之后不需要等待任何服务器的响应，可以得到最大吞吐量，但是消息丢失无法得知。
  - -1/all：消息发送后，需要等待所有副本都写入消息后才能收到服务器的响应，可以达到最强的可靠性。
- retries：请求发送失败，生产者会重试，设置为0禁止重试。
- batch.size：缓冲区大小
- linger.ms：生产者发送请求前等待一段时间，希望更多请求进入缓冲区。
- buffer.memory：生产者可用的缓存大小。
- key.serializer、value.serializer：将key和value对象ProducerRecord转换为字节。（broker端需要以`byte[]`形式存在）
- max.request.size：限制生产者客户端能发送消息的最大值。默认值为1MB。
- connections.max.idle.ms：指定多久之后关闭限制的连接。
- compression.type: 压缩方式，默认为none

实际使用过程中，可以使用`ProducerConfig`类来对Producer进行配置。


#### 发送过程
1. 连接到任意broker，获取集群元数据
2. 发送消息到指定分区leader副本所在的broker
3. 其他broker上的副本向leader同步数据



#### 消息对象字段

`ProducerRecord`作为消息对象，包含以下字段：

- topic：主题
- partition：分区号
- headers：消息头部，用于设定相关信息
- key：指定消息的键，可以用来计算分区号从而发送给特定分区**实现分类功能。**
  - 同一个key的消息会被划分到同一个分区。

- value：消息体
- timestamp：消息时间戳





#### 发送模式

- fire-and-forget：发后即忘，发送后不关心消息是否正确到达，可能会导致消息丢失。
  - 适用场景：只关心消息的吞吐量，并允许少量消息发送失败。配合参数`acks = 0`使用

- sync：同步发送，可以通过在`send()`后链式调用`get()`等到Kafka响应。
  - 适用场景：业务要求消息必须按照**顺序发送**，并且数据只能存储在同一个Partition中。

- async：异步发送，`send()`为异步发送。
  - 可以在send方法中指定`Callback()`函数，在消息返回响应时调用。
  - 适用场景：要求知道是否发送成功，并对消息顺序不关心。



```java
//fire-and-forget 模式   
producer.send(record);

//sync 模式 调用future.get()
future = producer.send(record);
RecordMetadata metadata = future.get();

//async
Future&lt;RecordMetadata&gt; send(ProducerRecord&lt;K, V&gt; record, Callback callback);
```







#### 生产者类型

生产者主要有两种类型：

- 幂等生产者：要求多次交付结果一致。
  - 启动幂等，需要将`enable.idempotence`设置为`true`，并且`retries`会被默认配置为`Integer.MAX_VALUE`，acks默认配置为`all`。
- 事务生产者：允许将消息原子性地发送给多个分区。



#### 分区器

分区器用来确定消息发往的分区，如果指定了`partition`字段，则不需要分区器。**若未指定，通过消息的key来计算partition值**。默认的分区器`org.apache.kafka.clients.producer.internals.DefaultPartitioner`中的`partition`方法定义了分区逻辑如下：

- 如果key不为null，那么分区器对key使用`MurmurHash2`算法进行哈希，从而得到分区号。
- 如果key为null，消息将以轮询的方式发往主题内的各个可用分区。



选择分区的策略有以下几种方式：

- 轮询策略：顺序分配消息
- 消息key指定分区策略：上述hash方式
- 随机策略：随机发送到某个分区上
- 自定义策略：实现`org.apache.kafka.clients.producer.Partitioner`接口，重写 `partition`方法。



#### 生产者拦截器

可以在消息发送之前做一些准备工作，例如：按照某个规则过滤不符合要求的消息、修改消息的内容等发送前的操作。`ProducerInterceptor`为默认拦截器。通过重写`onSend`来实现对消息的拦截修改等操作





#### 生产者客户端架构

整个生产者客户端由两个线程协调运行，分别为主线程和Sender线程。其分工如下：

- 主线程：创建消息，通过拦截器，序列化器、分区器作用后缓存到`RecordAccumulator`中。
- Sender线程：负责从`RecordAccumulator`中获取消息并发送给Kafka服务端。



**RecordAccumulator**细节：

- 缓存消息以便Sender线程可以批量发送，减少网络资源消耗。
- 缓存大小通过`buffer.memory`来设置，默认为32MB。
- 如果缓存空间被全部使用，send方法会被阻塞，可通过`max.block.ms`来控制阻塞时间。
- 消息都会被追加到累加器中的`Deque&lt;ProducerBatch&gt;`中，写入追加到双端队列尾部，Sender获取消息并发送时，从队列头部读取。`ProducerBatch`中包含多个`ProducerRecord`。
- 内部有一个`BufferPool`来实现对缓存的复用，使用`batch.size`对大小进行指定，该部分主要对发送给Kafka服务端消息之前进行保存。
- 当`ProducerRecord`传入累加器中，会首先寻找与消息分区对应的Deque，并获取尾部的`ProducerBatch` ，判断是否还可写入，不可写入则创建新的`ProducerBatch`。
- 一个Batch默认大小为16KB。



#### 元数据

- 记录了：存在哪些主题、分区、分区的leader是哪个节点，副本节点是哪几个等元数据。
- 需要更新元数据时，会挑选出`leastLoadedNode`，然后向Node发出请求获取元数据，请求由Sender线程发出，会使用sync来保证线程安全。





### 消费者 API

消费者负责订阅Kafka中的`Topic`，并且从订阅的`Topic`上拉取消息。每个消费者都有一个对应的消费组，当消息发送到主题后，只会被投递给订阅它的消费组中的一个消费者。

当新消费者加入组中时，会通过分区分配策略去给消费者分配分区。



消费逻辑需要具备以下几个步骤：

1. 配置消费者客户端参数并创建消费者实例。
2. 订阅主题。
3. 拉取消息并消费。
4. 提交消费位移。
5. 关闭实例。



```java
@Test
    public void test01() {
        Properties props = new Properties();
        props.setProperty(&#34;bootstrap.servers&#34;, &#34;localhost:9092&#34;);
        props.setProperty(&#34;group.id&#34;, &#34;test&#34;);
        props.setProperty(&#34;key.deserializer&#34;, &#34;org.apache.kafka.common.serialization.StringDeserializer&#34;);
        props.setProperty(&#34;value.deserializer&#34;, &#34;org.apache.kafka.common.serialization.StringDeserializer&#34;);
        props.setProperty(&#34;client.id&#34;,&#34;consumer.client.id.demo&#34;);
        KafkaConsumer&lt;String, String&gt; consumer = new KafkaConsumer&lt;&gt;(props);
        consumer.subscribe(Arrays.asList(topic));
        while (true) {
            ConsumerRecords&lt;String, String&gt; records = consumer.poll(Duration.ofMillis(100));
            for (ConsumerRecord&lt;String, String&gt; record : records)
                System.out.printf(&#34;offset = %d, key = %s, value = %s%n&#34;, record.offset(), record.key(), record.value());
        }
    }
```





#### 配置参数

- bootstrap.servers：集群地址
- group.id：隶属的消费组名称，默认为&#34;&#34;
- key.deserializer 和 value.deserializer：与生产者对应
- client.id：消费者id
- fetch.min(max).bytes：消费者一次拉取的最小（大）数据量。
- max.poll.records：消费者一次拉取的最大消息数量。


#### 消费过程

1. 连接到任意Broker，获取Kafka集群元数据
2. 通过上一步的元数据，找到自己所属Coordinator所在的broker
3. 加入Consumer Group，获取分区消费方案
4. 获取相关分区消费进度，从上次消费的offset开始继续拉取消息
5. 提交消费进度到Coordinator



#### 常用方法



`KafkaConsumer`提供了`subscribe`方法来订阅主题，若多次调用，以最后一次作为消费的主题。

```java
@Override
    public void subscribe(Collection&lt;String&gt; topics) {
        subscribe(topics, new NoOpConsumerRebalanceListener());
    }
```

还提供了`assign()`方法订阅主题中的特定分区，参数为`Collection&lt;TopicPartition&gt;`

```java
public void assign(Collection&lt;TopicPartition&gt; partitions);
```

`TopicPartition`类有两个属性：topic和partition，分别代表分区所属的主题和自身的分区编号。



`partitionsFor`用于查询指定主题的元数据，传入`topic`，返回`List&lt;PartitionInfo&gt;`。

```java
public List&lt;PartitionInfo&gt; partitionsFor(String topic)
```



`PartitionInfo`包含如下字段：

```java
//主题名称
private final String topic;

//分区编号
private final int partition;

//leader节点的位置
private final Node leader;

//AR集合
private final Node[] replicas;

//ISR集合
private final Node[] inSyncReplicas;

//OSR集合
private final Node[] offlineReplicas;
```



提供了`unsubscribe`方法来取消主题的订阅，若将`subscribe`方法中参数设置为空集合，效果等同于取消订阅。

```java
public void unsubscribe()
```



#### 反序列化

生产者对数据进行序列化， 消费者端用反序列化来恢复数据，其中包括ByteBufferDeserializer、ByteArrayDeserializer、BytesDeserializer、DoubleDeserializer、FloatDeserializer、IntegerDeserializer、LongDeserializer、ShortDeserializer、StringDeserializer，分别提供不同的类型反序列化。



可以考虑Thrift、Protocol Buffer等通用序列化工具来实现。



#### 消费模式

- push：服务端主动将消息推送给消费者。
- pull：消费者主动向服务端发起请求来拉取消息。提供的为`poll()`，Kafka采用的此种方式。



为什么不使用Push：Push方式无法确定消费者的消费速度，并且推送效率是Broker进行控制的，容易发生**消息堆积**的情况。





对于`poll()`，若分区中没有可消费的消息，拉去的结果为空，需要传入超时时间，用来控制该方法的阻塞时间。



#### Consumer Group机制

为什么要设计`Consumer Group`？

- 当Topic数据量非常大的时候，凭单个Consumer线程消费十分缓慢，需要用扩展性较好的机制来保证消费进度，Group机制是Kafka提供的消费者机制。




#### ConsumerRecord

对于消费者取回的消息，封装成`ConsumerRecord`类，各字段含义如下：

```java
//主题
private final String topic;

//分区
private final int partition;

//偏移量
private final long offset;

//时间戳
private final long timestamp;

//时间戳类型，CreateTime 和LogAppendTime
private final TimestampType timestampType;

//key被序列化之后的大小
private final int serializedKeySize;

//value被序列化之后的大小
private final int serializedValueSize;

//消息头部
private final Headers headers;

//消息键值对
private final K key;
private final V value;

//领导者节点任期
private final Optional&lt;Integer&gt; leaderEpoch;
```



#### 位移提交

消息的`commited offset`用于表示消息在分区中的相应位置，同样有一个`consumed offset`用来保存消费者消费位置。一般情况下`position = commited offset = consumed offset &#43; 1`。自动提交可能造成重复消费和消息丢失的现象。

位移提交的动作是消费完所有拉取到的信息之后执行的，如果消费过程中出现了异常，在故障恢复之后，会发生重复消费的现象，consumer需要做幂等性保障。



Kafka中消费位移的提交方式是自动提交，由消费者客户端参数`enable.auto.commit`控制，默认为true，定期提交的周期时间由`auto.commit.interval.ms`控制，默认为5s。

消费者每隔5秒会将拉取到的每个分区中最大的消息位移进行提交。自动位移提交的动作是在`poll()`方法的逻辑里完成的，在每次真正向服务端发起拉取请求之前会检查是否可以进行位移提交，如果可以，那么就会提交上一次轮询的位移。

服务端将`commited offset`保存在`__consumer_offsets`，它是由Kafka自动创建的，和普通的Topic存储格式相同。




#### 消费者拦截器



功能类似于生产者拦截器，可自定义。



#### 多线程实现



`KafkaConsumer`是非线程安全的，`acquire`检测当前是否只有一个线程在操作，若有其他线程则会抛出`ConcurrentModificationException`。



由于Kafka消息保留机制的作用，有些消息消费前可能被清理，可以通过线程封闭的方式来实现多线程消费：每个线程实例化一个`KafkaConsumer`对象。


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%B8%80-%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8/  

