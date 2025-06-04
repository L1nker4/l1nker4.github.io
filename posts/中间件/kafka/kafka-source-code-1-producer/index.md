# Kafka源码分析(一)- Producer生产逻辑


## 前言

clients模块是Kafka官方提供的默认Java客户端，该模块分为三部分：
1. Admin：提供了管理topic、partition、config的相关API
2. Consumer：提供了消费topic的API
3. Producer：提供了向topic投递消息的功能

源码以Kafka 3.9为例。

## Producer

Producer作为Kafka client中的消息生产者，提供send()方法用于写入消息，并有后台sender线程定时发送消息。



### send流程

核心send方法提供了以下两种异步方式：

```java
/**  
 * See {@link KafkaProducer#send(ProducerRecord)}  
 */Future&lt;RecordMetadata&gt; send(ProducerRecord&lt;K, V&gt; record);  
  
/**  支持回调方法
 * See {@link KafkaProducer#send(ProducerRecord, Callback)}  
 */Future&lt;RecordMetadata&gt; send(ProducerRecord&lt;K, V&gt; record, Callback callback);
```





#### interceptor机制

send流程中首先会检查用户是否自定义interceptor实现，用于处理send前置逻辑，具体业务场景不多赘述。

```java
@Override  
public Future&lt;RecordMetadata&gt; send(ProducerRecord&lt;K, V&gt; record, Callback callback) {  
    // 拦截器前置send动作  
    ProducerRecord&lt;K, V&gt; interceptedRecord = this.interceptors.onSend(record);  
    return doSend(interceptedRecord, callback);  
}
```

其中ProducerInterceptor提供了以下接口：
- ProducerRecord&lt;K, V&gt; onSend(ProducerRecord&lt;K, V&gt; record)：send前置处理逻辑
- onAcknowledgement(RecordMetadata metadata, Exception exception)：消息被应答之后或发送消息失败时调用。
- close()：用于关闭interceptor资源。


自定义interceptor需要考虑线程安全。


#### doSend流程

```java
private Future&lt;RecordMetadata&gt; doSend(ProducerRecord&lt;K, V&gt; record, Callback callback) {  
    // 1.1 创建callback对象  
    AppendCallbacks appendCallbacks = new AppendCallbacks(callback, this.interceptors, record);  
  
    try {  
        //1.2 检查producer是否被close  
        throwIfProducerClosed();  
        // first make sure the metadata for the topic is available  
        long nowMs = time.milliseconds();  
        ClusterAndWaitTime clusterAndWaitTime;  
  
        //1.3 拉取指定topic、分区的元数据，和等待时间  
        try {  
            clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);  
        } catch (KafkaException e) {  
            if (metadata.isClosed())  
                throw new KafkaException(&#34;Producer closed while send in progress&#34;, e);  
            throw e;  
        }  
        nowMs &#43;= clusterAndWaitTime.waitedOnMetadataMs;  
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);  
        Cluster cluster = clusterAndWaitTime.cluster;  
  
        //1.4 key value进行序列化  
        byte[] serializedKey;  
        try {  
            serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());  
        } catch (ClassCastException cce) {  
            throw new SerializationException(&#34;Can&#39;t convert key of class &#34; &#43; record.key().getClass().getName() &#43;  
                    &#34; to class &#34; &#43; producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() &#43;  
                    &#34; specified in key.serializer&#34;, cce);  
        }  
        byte[] serializedValue;  
        try {  
            serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());  
        } catch (ClassCastException cce) {  
            throw new SerializationException(&#34;Can&#39;t convert value of class &#34; &#43; record.value().getClass().getName() &#43;  
                    &#34; to class &#34; &#43; producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() &#43;  
                    &#34; specified in value.serializer&#34;, cce);  
        }  
  
        // 1.5 计算当前消息所属的partition  
        int partition = partition(record, serializedKey, serializedValue, cluster);  
  
        // 1.6 设置消息header为readOnly  
        setReadOnly(record.headers());  
        Header[] headers = record.headers().toArray();  
  
        //1.7 检查消息大小是否符合  
        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),  
                compression.type(), serializedKey, serializedValue, headers);  
        ensureValidRecordSize(serializedSize);  
        long timestamp = record.timestamp() == null ? nowMs : record.timestamp();  
  
        // 自定义partitioner  
        boolean abortOnNewBatch = partitioner != null;  
  
        // 1.8 将消息追加到accumulator中  
        RecordAccumulator.RecordAppendResult result = accumulator.append(record.topic(), partition, timestamp, serializedKey,  
                serializedValue, headers, appendCallbacks, remainingWaitMs, abortOnNewBatch, nowMs, cluster);  
        assert appendCallbacks.getPartition() != RecordMetadata.UNKNOWN_PARTITION;  
  
        // 1.9 消息入新batch的情况  
        if (result.abortForNewBatch) {  
            int prevPartition = partition;  
            onNewBatch(record.topic(), cluster, prevPartition);  
            partition = partition(record, serializedKey, serializedValue, cluster);  
            if (log.isTraceEnabled()) {  
                log.trace(&#34;Retrying append due to new batch creation for topic {} partition {}. The old partition was {}&#34;, record.topic(), partition, prevPartition);  
            }  
            result = accumulator.append(record.topic(), partition, timestamp, serializedKey,  
                serializedValue, headers, appendCallbacks, remainingWaitMs, false, nowMs, cluster);  
        }  
  
        // 2.1 开启事务的情况  
        if (transactionManager != null) {  
            transactionManager.maybeAddPartition(appendCallbacks.topicPartition());  
        }  
  
        // 2.2 如果batch满了，或者新batch被创建，唤醒后台sender线程  
        if (result.batchIsFull || result.newBatchCreated) {  
            log.trace(&#34;Waking up the sender since topic {} partition {} is either full or getting a new batch&#34;, record.topic(), appendCallbacks.getPartition());  
            this.sender.wakeup();  
        }  
        return result.future;  
        // handling exceptions and record the errors;  
        // for API exceptions return them in the future,        // for other exceptions throw directly    } catch (ApiException e) {  
        log.debug(&#34;Exception occurred during message send:&#34;, e);  
        if (callback != null) {  
            TopicPartition tp = appendCallbacks.topicPartition();  
            RecordMetadata nullMetadata = new RecordMetadata(tp, -1, -1, RecordBatch.NO_TIMESTAMP, -1, -1);  
            callback.onCompletion(nullMetadata, e);  
        }  
        this.errors.record();  
        this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);  
        if (transactionManager != null) {  
            transactionManager.maybeTransitionToErrorState(e);  
        }  
        return new FutureFailure(e);  
    } catch (InterruptedException e) {  
        this.errors.record();  
        this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);  
        throw new InterruptException(e);  
    } catch (KafkaException e) {  
        this.errors.record();  
        this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);  
        throw e;  
    } catch (Exception e) {  
        // we notify interceptor about all exceptions, since onSend is called before anything else in this method  
        this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);  
        throw e;  
    }  
}
```


####  partition机制

Producer中计算消息partition的流程较为简单：

1. 若record指定partition，则直接返回。
2. 若自定义了partitioner则使用自定义规则的分区计算方式
3. 若指定了key并未配置`partitioner.ignore.keys`，则使用murmur2算法得出partition
4. 否则将partition设置为UNKNOWN_PARTITION，这会在org.apache.kafka.clients.producer.internals.RecordAccumulator#append方法中进行处理。

```java
private int partition(ProducerRecord&lt;K, V&gt; record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {  
    if (record.partition() != null)  
        return record.partition();  
  
    if (partitioner != null) {  
        int customPartition = partitioner.partition(  
            record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);  
        if (customPartition &lt; 0) {  
            throw new IllegalArgumentException(String.format(  
                &#34;The partitioner generated an invalid partition number: %d. Partition number should always be non-negative.&#34;, customPartition));  
        }  
        return customPartition;  
    }  
  
    if (serializedKey != null &amp;&amp; !partitionerIgnoreKeys) {  
        // hash the keyBytes to choose a partition  
        return BuiltInPartitioner.partitionForKey(serializedKey, cluster.partitionsForTopic(record.topic()).size());  
    } else {  
        return RecordMetadata.UNKNOWN_PARTITION;  
    }  
}
```


Partitioner提供了以下三个接口：
1. int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster)：自定义分区计算方法。
2. void close()：用于partitioner关闭资源的方法。
3. void onNewBatch(String topic, Cluster cluster, int prevPartition)：从3.3.0开始废弃，用于通知partitioner新分区被创建，sticky分区方式可以改变新分区的黏性分区。


需要注意的是， [KIP-794](https://cwiki.apache.org/confluence/display/KAFKA/KIP-794%3A&#43;Strictly&#43;Uniform&#43;Sticky&#43;Partitioner)中指出，sticky分区方式会将消息发送给更慢的broker，慢broker因此收到更多的消息，逐渐变得更慢，因此在该提案中，做了如下更新：
1. partitioner默认配置设置为null，并且DefaultPartitioner和UniformStickyPartitioner都被废弃
2. 添加新配置用于分区计算

其他具体细节见提案。


#### RecordAccumulator写入流程

RecordAccumulator是Producer用于存储batch的cache，当达到一定阈值后，会由sender线程将消息发送到Kafka broker。

首先看下核心append方法的逻辑：

```java
/**  
 * Add a record to the accumulator, return the append result * &lt;p&gt;  
 * The append result will contain the future metadata, and flag for whether the appended batch is full or a new batch is created  
 * &lt;p&gt;  
 *  
 * @param topic The topic to which this record is being sent  
 * @param partition The partition to which this record is being sent or RecordMetadata.UNKNOWN_PARTITION  
 *                  if any partition could be used * @param timestamp The timestamp of the record  
 * @param key The key for the record  
 * @param value The value for the record  
 * @param headers the Headers for the record  
 * @param callbacks The callbacks to execute  
 * @param maxTimeToBlock The maximum time in milliseconds to block for buffer memory to be available  
 * @param abortOnNewBatch A boolean that indicates returning before a new batch is created and  
 *                        running the partitioner&#39;s onNewBatch method before trying to append again * @param nowMs The current time, in milliseconds  
 * @param cluster The cluster metadata  
 */public RecordAppendResult append(String topic,  
                                 int partition,  
                                 long timestamp,  
                                 byte[] key,  
                                 byte[] value,  
                                 Header[] headers,  
                                 AppendCallbacks callbacks,  
                                 long maxTimeToBlock,  
                                 boolean abortOnNewBatch,  
                                 long nowMs,  
                                 Cluster cluster) throws InterruptedException {  
  
    // 1.1 获取对应的topicInfo  
    TopicInfo topicInfo = topicInfoMap.computeIfAbsent(topic, k -&gt; new TopicInfo(createBuiltInPartitioner(logContext, k, batchSize)));  
  
    // We keep track of the number of appending thread to make sure we do not miss batches in  
    // abortIncompleteBatches().    appendsInProgress.incrementAndGet();  
    ByteBuffer buffer = null;  
    if (headers == null) headers = Record.EMPTY_HEADERS;  
    try {  
        // 2. while循环处理并发竟态情况  
        while (true) {  
            // 2.1 partition取值兜底  
            final BuiltInPartitioner.StickyPartitionInfo partitionInfo;  
            final int effectivePartition;  
            if (partition == RecordMetadata.UNKNOWN_PARTITION) {  
                //2.1.1 若未指定分区，使用粘性分区（默认0）  
                partitionInfo = topicInfo.builtInPartitioner.peekCurrentPartitionInfo(cluster);  
                effectivePartition = partitionInfo.partition();  
            } else {  
                partitionInfo = null;  
                effectivePartition = partition;  
            }  
  
            // 2.2 更新callback中的partition  
            setPartition(callbacks, effectivePartition);  
  
            // 2.3 获取当前分区的deque  
            Deque&lt;ProducerBatch&gt; dq = topicInfo.batches.computeIfAbsent(effectivePartition, k -&gt; new ArrayDeque&lt;&gt;());  
            synchronized (dq) {  
                // 2.4 check partition是否发生变化  
                if (partitionChanged(topic, topicInfo, partitionInfo, dq, nowMs, cluster))  
                    continue;  
  
                // 2.5 调用tryAppend进行写入  
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callbacks, dq, nowMs);  
  
                // 2.5.1 写入后result不为空，更新分区信息，细节见updatePartitionInfo  
                if (appendResult != null) {  
                    // If queue has incomplete batches we disable switch (see comments in updatePartitionInfo).  
                    boolean enableSwitch = allBatchesFull(dq);  
                    topicInfo.builtInPartitioner.updatePartitionInfo(partitionInfo, appendResult.appendedBytes, cluster, enableSwitch);  
                    return appendResult;  
                }  
            }  
  
            // 2.6 传入abortOnNewBatch为true，直接返回空batch，再次执行append进行写入  
            if (abortOnNewBatch) {  
                // Return a result that will cause another call to append.  
                return new RecordAppendResult(null, false, false, true, 0);  
            }  
  
            //2.8 buffer为空 分配空间并更新timestamp  
            if (buffer == null) {  
                byte maxUsableMagic = apiVersions.maxUsableProduceMagic();  
                int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression.type(), key, value, headers));  
                log.trace(&#34;Allocating a new {} byte message buffer for topic {} partition {} with remaining timeout {}ms&#34;, size, topic, effectivePartition, maxTimeToBlock);  
                // This call may block if we exhausted buffer space.  
                buffer = free.allocate(size, maxTimeToBlock);  
                // Update the current time in case the buffer allocation blocked above.  
                // NOTE: getting time may be expensive, so calling it under a lock                // should be avoided.                nowMs = time.milliseconds();  
            }  
  
            //3. 如果上轮deque为空，且abortOnNewBatch=false，则尝试重新将消息写入新batch  
            synchronized (dq) {  
                // After taking the lock, validate that the partition hasn&#39;t changed and retry.  
                if (partitionChanged(topic, topicInfo, partitionInfo, dq, nowMs, cluster))  
                    continue;  
  
                //3.1 将新batch加到deque，将消息加到新batch  
                RecordAppendResult appendResult = appendNewBatch(topic, effectivePartition, dq, timestamp, key, value, headers, callbacks, buffer, nowMs);  
                // Set buffer to null, so that deallocate doesn&#39;t return it back to free pool, since it&#39;s used in the batch.  
                if (appendResult.newBatchCreated)  
                    buffer = null;  
                // If queue has incomplete batches we disable switch (see comments in updatePartitionInfo).  
                boolean enableSwitch = allBatchesFull(dq);  
                topicInfo.builtInPartitioner.updatePartitionInfo(partitionInfo, appendResult.appendedBytes, cluster, enableSwitch);  
                return appendResult;  
            }  
        }  
    } finally {  
        //4. 释放buffer，并且减少appendsInProgress  
        free.deallocate(buffer);  
        appendsInProgress.decrementAndGet();  
    }  
}
```



从中可以看出RecordAccumulator的一个核心属性：
```java
//topicInfoMap是由topic到TopicInfo属性的映射
private final ConcurrentMap&lt;String /*topic*/, TopicInfo&gt; topicInfoMap = new CopyOnWriteMap&lt;&gt;();
```


内置类TopicInfo：

```java
private static class TopicInfo {
	//分区到deque的映射，deque由ProducerBatch构成
    public final ConcurrentMap&lt;Integer /*partition*/, Deque&lt;ProducerBatch&gt;&gt; batches = new CopyOnWriteMap&lt;&gt;();  

	//内置partitioner，KIP-794更新
    public final BuiltInPartitioner builtInPartitioner;  
  
    public TopicInfo(BuiltInPartitioner builtInPartitioner) {  
        this.builtInPartitioner = builtInPartitioner;  
    }  
}
```

上述结构来看，写入的分区由ProducerBatch队列构成，ProducerBatch写入的核心方法tryAppend()使用MemoryRecordsBuilder执行写入，并处理压缩、格式转换等。

```java
/**  
 *  Try to append to a ProducerBatch. * *  If it is full, we return null and a new batch is created. We also close the batch for record appends to free up *  resources like compression buffers. The batch will be fully closed (ie. the record batch headers will be written *  and memory records built) in one of the following cases (whichever comes first): right before send, *  if it is expired, or when the producer is closed. */private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers,  
                                     Callback callback, Deque&lt;ProducerBatch&gt; deque, long nowMs) {  
    if (closed)  
        throw new KafkaException(&#34;Producer closed while send in progress&#34;);  
    //1. 获取deque最后一个batch  
    ProducerBatch last = deque.peekLast();  
    if (last != null) {  
        //2. 获取当前batch的size  
        int initialBytes = last.estimatedSizeInBytes();  
        //3. 尝试将消息加到batch  
        FutureRecordMetadata future = last.tryAppend(timestamp, key, value, headers, callback, nowMs);  
  
        //4. 如果batch已满，关闭batch，返回null  
        if (future == null) {  
            last.closeForRecordAppends();  
        } else {  
            //5. 计算写入的消息大小，并返回RecordAppendResult  
            int appendedBytes = last.estimatedSizeInBytes() - initialBytes;  
            return new RecordAppendResult(future, deque.size() &gt; 1 || last.isFull(), false, false, appendedBytes);  
        }  
    }  
    //deque 为空，return null  
    return null;  
}
```


```java
/**  
 * Append the record to the current record set and return the relative offset within that record set * * @return The RecordSend corresponding to this record or null if there isn&#39;t sufficient room.  
 */public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers, Callback callback, long now) {  
    //1.1 检查MemoryRecordsBuilder是否还有足够空间用于写入  
    if (!recordsBuilder.hasRoomFor(timestamp, key, value, headers)) {  
        //1.1.1 没有空间写入直接return null  
        return null;  
    } else {  
        //1.2 调用append()方法写入消息，更新对应字段并return future  
        this.recordsBuilder.append(timestamp, key, value, headers);  
        this.maxRecordSize = Math.max(this.maxRecordSize, AbstractRecords.estimateSizeInBytesUpperBound(magic(),  
                recordsBuilder.compression().type(), key, value, headers));  
        this.lastAppendTime = now;  
        //1.3 创建返回值  
        FutureRecordMetadata future = new FutureRecordMetadata(this.produceFuture, this.recordCount,  
                                                               timestamp,  
                                                               key == null ? -1 : key.length,  
                                                               value == null ? -1 : value.length,  
                                                               Time.SYSTEM);  
        // we have to keep every future returned to the users in case the batch needs to be  
        // split to several new batches and resent.        thunks.add(new Thunk(callback, future));  
        this.recordCount&#43;&#43;;  
        return future;  
    }  
}
```


MemoryRecordsBuilder执行写入会检查消息格式，并区分不同版本的消息写入方式。

```java

private void appendWithOffset(long offset, boolean isControlRecord, long timestamp, ByteBuffer key,  
                              ByteBuffer value, Header[] headers) {  
    try {  
        //1. 检查isControl标志是否一致  
        if (isControlRecord != isControlBatch)  
            throw new IllegalArgumentException(&#34;Control records can only be appended to control batches&#34;);  
  
        // 2. 检查offset递增  
        if (lastOffset != null &amp;&amp; offset &lt;= lastOffset)  
            throw new IllegalArgumentException(String.format(&#34;Illegal offset %d following previous offset %d &#34; &#43;  
                    &#34;(Offsets must increase monotonically).&#34;, offset, lastOffset));  
  
        //3. 检查时间戳  
        if (timestamp &lt; 0 &amp;&amp; timestamp != RecordBatch.NO_TIMESTAMP)  
            throw new IllegalArgumentException(&#34;Invalid negative timestamp &#34; &#43; timestamp);  
  
        //4. 只有V2版本消息，才有header  
        if (magic &lt; RecordBatch.MAGIC_VALUE_V2 &amp;&amp; headers != null &amp;&amp; headers.length &gt; 0)  
            throw new IllegalArgumentException(&#34;Magic v&#34; &#43; magic &#43; &#34; does not support record headers&#34;);  
  
        if (baseTimestamp == null)  
            baseTimestamp = timestamp;  
  
        //5. 写入  
        if (magic &gt; RecordBatch.MAGIC_VALUE_V1) {  
            appendDefaultRecord(offset, timestamp, key, value, headers);  
        } else {  
            appendLegacyRecord(offset, timestamp, key, value, magic);  
        }  
    } catch (IOException e) {  
        throw new KafkaException(&#34;I/O exception when writing to the append stream, closing&#34;, e);  
    }  
}
```


appendDefaultRecord()方法负责写入消息到stream流中，并更新元信息。

```java
private void appendDefaultRecord(long offset, long timestamp, ByteBuffer key, ByteBuffer value,  
                                 Header[] headers) throws IOException {  
    //1. 检查appendStream状态  
    ensureOpenForRecordAppend();  
    //2. 计算各个变量值  
    int offsetDelta = (int) (offset - baseOffset);  
    long timestampDelta = timestamp - baseTimestamp;  
  
    //3. 调用DefaultRecord类的writeTo方法，将消息写入appendStream流  
    int sizeInBytes = DefaultRecord.writeTo(appendStream, offsetDelta, timestampDelta, key, value, headers);  
    //更新元信息  
    recordWritten(offset, timestamp, sizeInBytes);  
}
```


### Sender线程

sender在KafkaProducer构造方法中初始化：
```java
this.sender = newSender(logContext, kafkaClient, this.metadata);  
String ioThreadName = NETWORK_THREAD_PREFIX &#43; &#34; | &#34; &#43; clientId;  
this.ioThread = new KafkaThread(ioThreadName, this.sender, true);  
this.ioThread.start();
```


Sender是一个Runnable对象，核心逻辑如下：
```java
while (running) {  
    try {  
        runOnce();  
    } catch (Exception e) {  
        log.error(&#34;Uncaught error in kafka producer I/O thread: &#34;, e);  
    }  
}
```

runOnce()通过sendProducerData()执行实际的发送逻辑，最后通过poll()方法处理网络IO请求

```java
void runOnce() {  
    ……
    //省略事务处理

	//创建发送给broker的请求并发送
    long currentTimeMs = time.milliseconds();
    long pollTimeout = sendProducerData(currentTimeMs);
    //处理实际网络IO socket的入口，负责发送请求、接收响应
    client.poll(pollTimeout, currentTimeMs);
}
```


#### sendProducerData流程

该方法主干逻辑如下：
1. 调用RecordAccumulator的ready()方法获取可以发送的Node消息
2. 调用RecordAccumulator的drain()，获取nodeId -&gt; 待发送的ProducerBatch集合映射
3. 调用sendProduceRequests()按Node分组发送请求

其中guaranteeMessageOrder取决于`max.in.flight.requests.per.connection`配置是否等于1

```java
private long sendProducerData(long now) {  
    MetadataSnapshot metadataSnapshot = metadata.fetchMetadataSnapshot();  
    // 1.1 查询accumulator的ready()方法，获取当前已经满足发送要求的node（只需要有一个batch满足发送要求）  
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(metadataSnapshot, now);  
  
    // 1.2 如果metadata中有topic的partition leader未知，先更新metadata  
    if (!result.unknownLeaderTopics.isEmpty()) {  
        // The set of topics with unknown leader contains topics with leader election pending as well as  
        // topics which may have expired. Add the topic again to metadata to ensure it is included        // and request metadata update, since there are messages to send to the topic.        for (String topic : result.unknownLeaderTopics)  
            this.metadata.add(topic, now);  
  
        log.debug(&#34;Requesting metadata update due to unknown leader topics from the batched records: {}&#34;,  
            result.unknownLeaderTopics);  
        this.metadata.requestUpdate(false);  
    }  
  
    // 1.3 删除暂未connection ready的node  
    Iterator&lt;Node&gt; iter = result.readyNodes.iterator();  
    long notReadyTimeout = Long.MAX_VALUE;  
    while (iter.hasNext()) {  
        Node node = iter.next();  
        //1.4 更新readyTimeMs  
        if (!this.client.ready(node, now)) {  
            // Update just the readyTimeMs of the latency stats, so that it moves forward  
            // every time the batch is ready (then the difference between readyTimeMs and            // drainTimeMs would represent how long data is waiting for the node).            this.accumulator.updateNodeLatencyStats(node.id(), now, false);  
            iter.remove();  
            notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));  
        } else {  
            // Update both readyTimeMs and drainTimeMs, this would &#34;reset&#34; the node  
            // latency.            this.accumulator.updateNodeLatencyStats(node.id(), now, true);  
        }  
    }  
  
    // 2.1 调用drain()，获取nodeId -&gt; 待发送的ProducerBatch集合映射  
    Map&lt;Integer, List&lt;ProducerBatch&gt;&gt; batches = this.accumulator.drain(metadataSnapshot, result.readyNodes, this.maxRequestSize, now);  
  
    //2.2 调用addToInflightBatches()，将待发送的ProducerBatch集合映射添加到inFlightBatches中，这个集合记录了已经发送但未响应的ProducerBatch  
    addToInflightBatches(batches);  
  
    //2.3 如果guaranteeMessageOrder为true，将batch添加到muted  
    if (guaranteeMessageOrder) {  
        // Mute all the partitions drained  
        for (List&lt;ProducerBatch&gt; batchList : batches.values()) {  
            for (ProducerBatch batch : batchList)  
                this.accumulator.mutePartition(batch.topicPartition);  
        }  
    }  
  
    //2.4 重置batch到期时间  
    accumulator.resetNextBatchExpiryTime();  
  
    //2.5 获取过期的batch  
    List&lt;ProducerBatch&gt; expiredInflightBatches = getExpiredInflightBatches(now);  
    List&lt;ProducerBatch&gt; expiredBatches = this.accumulator.expiredBatches(now);  
    expiredBatches.addAll(expiredInflightBatches);  
  
    //2.6 循环调用failBatch()方法来处理过期的batch，内部调用ProducerBatch.done()  
    if (!expiredBatches.isEmpty())  
        log.trace(&#34;Expired {} batches in accumulator&#34;, expiredBatches.size());  
    for (ProducerBatch expiredBatch : expiredBatches) {  
        String errorMessage = &#34;Expiring &#34; &#43; expiredBatch.recordCount &#43; &#34; record(s) for &#34; &#43; expiredBatch.topicPartition  
            &#43; &#34;:&#34; &#43; (now - expiredBatch.createdMs) &#43; &#34; ms has passed since batch creation&#34;;  
        failBatch(expiredBatch, new TimeoutException(errorMessage), false);  
        if (transactionManager != null &amp;&amp; expiredBatch.inRetry()) {  
            // This ensures that no new batches are drained until the current in flight batches are fully resolved.  
            transactionManager.markSequenceUnresolved(expiredBatch);  
        }  
    }  
  
    //2.7 更新metric  
    sensors.updateProduceRequestMetrics(batches);  
      
    //2.8 计算pollTimeout  
    long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);  
    pollTimeout = Math.min(pollTimeout, this.accumulator.nextExpiryTimeMs() - now);  
    pollTimeout = Math.max(pollTimeout, 0);  
    if (!result.readyNodes.isEmpty()) {  
        log.trace(&#34;Nodes with data ready to send: {}&#34;, result.readyNodes);  
        // if some partitions are already ready to be sent, the select time would be 0;  
        // otherwise if some partition already has some data accumulated but not ready yet,        // the select time will be the time difference between now and its linger expiry time;        // otherwise the select time will be the time difference between now and the metadata expiry time;        pollTimeout = 0;  
    }  
    //2.9 调用sendProduceRequests()按Node分组发送请求  
    sendProduceRequests(batches, now);  
    return pollTimeout;  
}
```


#### ready流程

从batchReady()方法中可以看出，是否被确定为ready node，只需要满足以下几个条件中的任何一条：

- full：full = dequeSize &gt; 1 || batch.isFull()
- expired：当前等待时间是否大于lingerMs，其中有重试backoff参数的影响
- exhausted：buffer pool是否已满
- closed：accumulator是否被关闭
- flushInProgress()：是否有其他线程在调用flush()，见：org.apache.kafka.clients.producer.KafkaProducer#flush
- transactionCompleting：若开启事务，且事务正准备完成

```java
/**  
 * Add the leader to the ready nodes if the batch is ready * * @param exhausted &#39;true&#39; is the buffer pool is exhausted  
 * @param part The partition  
 * @param leader The leader for the partition  
 * @param waitedTimeMs How long batch waited  
 * @param backingOff Is backing off  
 * @param backoffAttempts Number of attempts for calculating backoff delay  
 * @param full Is batch full  
 * @param nextReadyCheckDelayMs The delay for next check  
 * @param readyNodes The set of ready nodes (to be filled in)  
 * @return The delay for next check  
 */private long batchReady(boolean exhausted, TopicPartition part, Node leader,  
                        long waitedTimeMs, boolean backingOff, int backoffAttempts,  
                        boolean full, long nextReadyCheckDelayMs, Set&lt;Node&gt; readyNodes) {  
    if (!readyNodes.contains(leader) &amp;&amp; !isMuted(part)) {  
        long timeToWaitMs = backingOff ? retryBackoff.backoff(backoffAttempts &gt; 0 ? backoffAttempts - 1 : 0) : lingerMs;  
        boolean expired = waitedTimeMs &gt;= timeToWaitMs;  
        boolean transactionCompleting = transactionManager != null &amp;&amp; transactionManager.isCompleting();  
        boolean sendable = full  
                || expired  
                || exhausted  
                || closed  
                || flushInProgress()  
                || transactionCompleting;  
        if (sendable &amp;&amp; !backingOff) {  
            readyNodes.add(leader);  
        } else {  
            long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);  
            // Note that this results in a conservative estimate since an un-sendable partition may have  
            // a leader that will later be found to have sendable data. However, this is good enough            // since we&#39;ll just wake up and then sleep again for the remaining time.            nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);  
        }  
    }  
    return nextReadyCheckDelayMs;  
}
```


#### sendProduceRequests

sendProduceRequests()会基于每个node去请求。

```java
/**  
 * Transfer the record batches into a list of produce requests on a per-node basis */private void sendProduceRequests(Map&lt;Integer, List&lt;ProducerBatch&gt;&gt; collated, long now) {  
    for (Map.Entry&lt;Integer, List&lt;ProducerBatch&gt;&gt; entry : collated.entrySet())  
        sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue());  
}
```



## Reference

[https://docs.confluent.io/kafka-client/overview.html](https://docs.confluent.io/kafka-client/overview.html)
[https://learn.conduktor.io/kafka/kafka-producers-advanced](https://learn.conduktor.io/kafka/kafka-producers-advanced)

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-1-producer/  

