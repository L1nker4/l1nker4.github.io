# Kafka学习笔记(二)-存储架构




# Kafka存储架构



Kafka是为了解决大数据量的实时日志流而产生的，日志流主要特点包括：

1. 数据实时存储
2. 海量数据存储与处理



Kafka需要保证以下几点：

- 存储的主要是消息流
- 要保证海量数据的高效存储
- 要支持海量数据的高效检索
- 要保证数据的安全性和稳定性



Kafka使用的存储方案是：磁盘顺序写 &#43; 稀疏哈希索引。



## 日志目录布局



Kafka中消息以Topic为单位归类，各个Topic下面分为多个分区，分区中每条消息都会被分配一个唯一的序列号（offset）。日志命名方式为：`&lt;topic&gt;-&lt;partition&gt;`

在不考虑多副本的情况下，一个分区对应一个Log，为了防止Log过大，Kafka引入`LogSegment`，将Log切分为多个`LogSegment`。其结构如下所示：

- Log
  - LogSegment：每个 LogSegment 都有一个基准偏移量 baseOffset，用来表示当前 LogSegment中第一条消息的offset。只有最后一个LogSegment才能写入。下述文件根据`baseOffset`命名，长度固定为20位数字，没有达到的位数用0填充。
    - .log：日志文件
    - .index：偏移量索引文件
    - .timeindex：时间戳索引文件
    - .snapshot：快照索引文件



## 消息格式

消息格式关系到存储性能，比如冗余字段会增加分区的存储空间、网络传输的开销较大。



Kafka3.0中将`BatchRecords`作为磁盘中的存储单元，一个`BatchRecords`中包含多个`Record`。

`BatchRecords`的格式如下：

```
baseOffset: int64
batchLength: int32
partitionLeaderEpoch: int32
magic: int8 (current magic value is 2)
crc: int32
attributes: int16
    bit 0~2:
        0: no compression
        1: gzip
        2: snappy
        3: lz4
        4: zstd
    bit 3: timestampType
    bit 4: isTransactional (0 means not transactional)
    bit 5: isControlBatch (0 means not a control batch)
    bit 6: hasDeleteHorizonMs (0 means baseTimestamp is not set as the delete horizon for compaction)
    bit 7~15: unused
lastOffsetDelta: int32
baseTimestamp: int64
maxTimestamp: int64
producerId: int64
producerEpoch: int16
baseSequence: int32
records: [Record]
```

字段解释如下：

|         字段         |                             含义                             |
| :------------------: | :----------------------------------------------------------: |
|      baseOffset      |                     这批消息的起始Offset                     |
| partitionLeaderEpoch |                用于分区恢复时保护数据的一致性                |
|     batchLength      |                      BatchRecords的长度                      |
|        magic         |    魔数，可以用于拓展存储一些信息，当前3.0版本的magic是2     |
|         crc          | crc校验码，包含从attributes开始到BatchRecords结束的数据的校验码 |
|      attributes      | int16，其中bit0-2中包含了使用的压缩算法，bit3是timestampType，bit4表示是否失误，bit5表示是否是控制指令，bit6-15暂未使用 |
|   lastOffsetDelta    |      BatchRecords中最后一个Offset，是相对baseOffset的值      |
|    firstTimestamp    |                BatchRecords中最小的timestamp                 |
|     maxTimestamp     |                BatchRecords中最大的timestamp                 |
|      producerId      |             发送端的唯一ID，用于做消息的幂等处理             |
|    producerEpoch     |             发送端的Epoch，用于做消息的幂等处理              |
|     baseSequence     |          BatchRecords的序列号，用于做消息的幂等处理          |
|       records        |                        具体的消息内容                        |


Record格式如下：

```
length: varint
attributes: int8
    bit 0~7: unused
timestampDelta: varlong
offsetDelta: varint
keyLength: varint
key: byte[]
valueLen: varint
value: byte[]
Headers =&gt; [Header]

headerKeyLength: varint
headerKey: String
headerValueLength: varint
Value: byte[]
```



上述使用的`Varint`编码方式，该编码起到很好的压缩效果。ProtoBuf使用的也是该编码方式。





### 消息压缩

压缩算法在生产者客户端进行配置，通过`compression.type`参数选择合适的算法，主要有`gzip`, `lz4`, `snappy`,`zstd`



## 日志索引



索引用来提高查找的效率，**偏移量索引文件**建立了offset到物理地址之间的映射关系，时间戳索引文件根据指定的时间戳来查找对应的偏移量信息。



Kafka中索引文件以**稀疏索引**的方式构造消息的索引，不能保证每个消息在索引文件中都有对应的索引项，每写入一定量的消息（由broker端的`log.index.interval.bytes`指定）时，两个索引文件分别增加一个偏移量索引项和时间戳索引项。

稀疏索引通过`MappedByteBuffer`将索引文件映射到内存中，以加快索引的查询速度。两个索引文件中索引项根据偏移量和时间戳单调递增的，可以使用**二分查找**快速定位到。



日志分段策略由以下配置决定（满足其一即可）：

- broker的`log.segment.bytes `：当前LogSegment大小超过该值，默认1GB
- 当前日志分段中消息的最大时间戳与当前系统的时间戳的差值大于 `log.roll.ms`或`log.roll.hours`参数配置的值。
- 偏移量索引文件或时间戳索引文件的大小达到broker端参数`log.index.size.max.bytes`配置的值。默认为10MB
- 追加的消息的偏移量与当前日志分段的偏移量之间的差值大于`Integer.MAX_VALUE`



### 偏移量索引项



每个索引项占用8个字节，分为以下两个部分：

- relativeOffset：相对偏移量，消息相对于`baseOffset`的偏移量，占用4个字节
- position：物理地址，消息在日志分段文件中的物理位置，占用4个字节。

Kafka要求索引文件大小必须是索引项的整数倍。



### 时间戳索引项

时间戳索引项占用12字节，分为以下两个部分：

- timestamp：当前日志分段最大的时间戳
  - 追加的索引项该字段必须大于前面的时间戳，保持单调递增属性
- relativeOffset：时间戳对应的相对偏移量



## 日志清理



Kafka将消息存储在磁盘上，为了控制占用空间的不断增加，需要对消息做一定的清理操作。提供了两种日志清理策略：

- 日志删除：按照一定的保留策略直接删除不符合条件的日志分段。
- 日志压缩：对相同key进行整合，保留最新版本的value。



通过broker端参数`log.cleanup.policy`来设置日志清理策略，默认采用`delete`策略，日志清理的粒度可以控制到主题级别



### 日志删除

日志管理器中会有一个日志删除任务来周期性检测并删除不符合保留条件的日志分段文件，周期通过`log.retention.check.interval.ms`来配置，默认五分钟，保留策略如下：

- 基于时间的保留策略：删除日志文件中超过设定阈值的日志分段
- 基于日志大小的保留策略：删除超过阈值的日志分段
- 基于日志起始偏移量的保留策略：某日志分段的下一个日志分段的起始偏移量baseOffset 是否小于等于logStartOffset，若是，则可以删除此日志分段。



### 日志压缩

`Log Compaction`前后，分段中每条消息的偏移量和写入时偏移量保持一致，日志中每条消息的物理位置会改变，类似Redis的RDB过程



## 磁盘存储



**磁盘顺序写速度比磁盘随机写快，比内存随机写快**。

Kafka采用append的方式写入消息，并且不允许修改写入的消息，这种方式属于**磁盘顺序写**。

### Page Cache

`Page Cache`是OS实现的一种主要的磁盘缓存，以此减少对磁盘IO的操作。具体流程：将磁盘数据缓存到内存，将对磁盘的访问转为对内存的访问，极大提高访问效率。



当进程访问磁盘时，会首先查看数据所在的页是否在`page cache`中，没有命中，则向磁盘发起读取请求并将数据存入缓存。



Kafka中大量使用了页缓存，这是Kafka实现高吞吐的重要因素之一，虽然刷盘由OS控制，Kafka同样提供了`fsync`强制刷盘来保证数据可靠性。



### 磁盘IO流程



磁盘IO的场景从编程角度分为以下几种：

- 调用C标准库，数据流向：进程buffer-&gt;C 标准IO库 buffer-&gt;page cache-&gt;磁盘上的具体文件
- 调用文件IO，数据流向：进程buffer-&gt;page cache -&gt;磁盘
- 打开文件使用`O_DIRECT`，直接读磁盘



磁盘IO流程如下：

- **写操作**：调用`fwrite`把数据写入 IO Buffer（会存在将多个请求打包在一个buffer）。调用`write`将数据写入页缓存，此时不会主动刷盘，存在`pdflush`线程不断检测脏页，并判断是否写回磁盘。
- **读操作**：调用`fread`去IO Buffer中读取数据，读取到成功返回，否则去页缓存读取，不存在去磁盘读。
- **IO请求处理**：通用块层将IO请求构造成多个bio结构提交给调度层，调度器将bio结构排序：**尽可能将随机读写变为顺序读写**



从上到下，逐层分别为：

- 应用层：应用程序
- 内核层：虚拟文件系统 &#43; 具体文件系统，接收应用层的内核调用，并并封装请求发送给通用块层。
- 通用块层：接受上层发出的磁盘请求，并调整结构传给设备层，该层隐藏了底层硬件块的特性，提供了通用视图。
- 设备层：磁盘



Linux的IO调度策略：

- NOOP：FIFO队列
- CFQ：按照IO请求地址进行排序
- DEADLINE：在CFQ的基础上，解决了IO请求饥饿问题。
- ANTICIPATORY：为每个读IO设置了6ms的等待时间窗口，如果在此期间内，收到了相邻位置的读IO请求，立即满足



Kafka还采用了**零拷贝**技术来提升性能，直接将数据从磁盘文件复制到网卡设备，不经过应用程序的进程空间，减少了内核态和用户态的切换次数，大大提升了性能，Java的NIO采用`FileChannal.transferTo()`实现零拷贝。


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%BA%8C-%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84/  

