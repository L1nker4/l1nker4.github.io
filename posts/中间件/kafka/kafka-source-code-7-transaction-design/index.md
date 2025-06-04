# Kafka源码分析(七) - 事务设计


# 1. Intro

Kafka事务主要适用于以下两种场景：

1. multi-produce场景：Producer需要将多批次消息进行原子性提交。
2. consume-transform-produce场景：消费上游数据后，经过处理，生产下游数据。该场景实现可以参考`kafka.examples.ExactlyOnceMessageProcessor`，事务可以保证消费和生产的原子性。

![consume-transform-produce场景](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250402110429.png)


consume-transform-produce场景案例：

```java
@Override  
public void run() {  
    int retries = 0;  
    int processedRecords = 0;  
    long remainingRecords = Long.MAX_VALUE;  
  
    // it is recommended to have a relatively short txn timeout in order to clear pending offsets faster  
    int transactionTimeoutMs = 10_000;  
    // consumer must be in read_committed mode, which means it won&#39;t be able to read uncommitted data  
    boolean readCommitted = true;  
  
    try (KafkaProducer&lt;Integer, String&gt; producer = new Producer(&#34;processor-producer&#34;, bootstrapServers, outputTopic,  
            true, transactionalId, true, -1, transactionTimeoutMs, null).createKafkaProducer();  
         KafkaConsumer&lt;Integer, String&gt; consumer = new Consumer(&#34;processor-consumer&#34;, bootstrapServers, inputTopic,  
             &#34;processor-group&#34;, Optional.of(groupInstanceId), readCommitted, -1, null).createKafkaConsumer()) {  
        // called first and once to fence zombies and abort any pending transaction  
        producer.initTransactions();  
        consumer.subscribe(singleton(inputTopic), this);  
  
        Utils.printOut(&#34;Processing new records&#34;);  
        while (!closed &amp;&amp; remainingRecords &gt; 0) {  
            try {  
                ConsumerRecords&lt;Integer, String&gt; records = consumer.poll(ofMillis(200));  
                if (!records.isEmpty()) {  
                    // begin a new transaction session  
                    producer.beginTransaction();  
  
                    for (ConsumerRecord&lt;Integer, String&gt; record : records) {  
                        // process the record and send downstream  
                        ProducerRecord&lt;Integer, String&gt; newRecord =  
                            new ProducerRecord&lt;&gt;(outputTopic, record.key(), record.value() &#43; &#34;-ok&#34;);  
                        producer.send(newRecord);  
                    }  
  
                    // checkpoint the progress by sending offsets to group coordinator broker  
                    // note that this API is only available for broker &gt;= 2.5                    
	            producer.sendOffsetsToTransaction(getOffsetsToCommit(consumer), consumer.groupMetadata());  
  
                    // commit the transaction including offsets  
                    producer.commitTransaction();  
                    processedRecords &#43;= records.count();  
                    retries = 0;  
                }  
            } catch (AuthorizationException | UnsupportedVersionException | ProducerFencedException  
                     | FencedInstanceIdException | OutOfOrderSequenceException | SerializationException e) {  
                // we can&#39;t recover from these exceptions  
                Utils.printErr(e.getMessage());  
                shutdown();  
            } catch (OffsetOutOfRangeException | NoOffsetForPartitionException e) {  
                // invalid or no offset found without auto.reset.policy  
                Utils.printOut(&#34;Invalid or no offset found, using latest&#34;);  
                consumer.seekToEnd(emptyList());  
                consumer.commitSync();  
                retries = 0;  
            } catch (KafkaException e) {  
                // abort the transaction  
                Utils.printOut(&#34;Aborting transaction: %s&#34;, e.getMessage());  
                producer.abortTransaction();  
                retries = maybeRetry(retries, consumer);  
            }  
  
            remainingRecords = getRemainingRecords(consumer);  
            if (remainingRecords != Long.MAX_VALUE) {  
                Utils.printOut(&#34;Remaining records: %d&#34;, remainingRecords);  
            }  
        }  
    } catch (Throwable e) {  
        Utils.printErr(&#34;Unhandled exception&#34;);  
        e.printStackTrace();  
    }  
    Utils.printOut(&#34;Processed %d records&#34;, processedRecords);  
    shutdown();  
}
```


## 1.1. 客户端配置项

Producer配置项：
1. `enable.idempotence`：默认false，开启后会设置acks=all, retries=Integer.MAX_VALUE,max.inflight.requests.per.connection=1
2. `transactional.id`：事务ID，用于标识事务，允许跨多个client


Consumer配置项：
1. `isolation.level`：默认值为`read_uncommitted`
	- read_uncommitted：允许消费暂未提交事务的message
	- read_commited：允许消费除未提交事务以外的message



# 2. 设计


## 2.1. 事务流程


![流程图](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250402165515.png)


1. client发送FindCoordinatorRequest请求给broker端，获取Transaction Coordinator服务地址。
2. client调用initializeTransactions方法发送InitProducerIdRequest请求给Transaction Coordinator，用以获取producerId。
3. client调用beginTransaction方法，开启一次事务，client本地会改变事务状态，并不会影响Transaction Coordinator服务。
4. 事务过程：
	1. client开启事务后，当新的TopicPartition写入数据时，Producer会向Transaction Coordinator发送AddPartitionsToTxnRequest，Transaction Coordinator记录对应的事务分区
	2. client向TopicPartition的leader endpoint发送ProduceRequest
	3. client调用sendOffsetsToTransaction方法向Transaction Coordinator发送AddOffsetsToTxnRequest，TC会将对应的消费记录存储到事务日志中。
	4. client完成向Transaction Coordinator发送消费记录后，client将会发送TxnOffsetCommitRequest给consumer coordinator
5. 事务提交 or 事务回滚
	1. 调用commitTransaction/abortTransaction方法，向Transaction Coordinator发送EndTxnRequest
	2. Transaction Coordinator向TopicPartition的leader endpoint发送WriteTxnMarkerRequest，Broker端会根据情况选择commit或rollback
	3. Transaction Coordinator将事务结果写入事务日志中


# 3. Producer端实现

## 3.1. 接口概述


Java客户端中KafkaProducer提供了以下接口，供用户实现事务需求：

```java
public interface Producer&lt;K, V&gt; extends Closeable {  

//初始化事务，申请producer id等事务字段
void initTransactions();  

//开启事务，改变事务状态
void beginTransaction() throws ProducerFencedException;  

//提交消费位置，offsets是每个分区的消费位置，consumerGroupId为消费组id，允许将consumer的消费进度和producer绑定在同一个事务
void sendOffsetsToTransaction(Map&lt;TopicPartition, OffsetAndMetadata&gt; offsets,  
						  ConsumerGroupMetadata groupMetadata) throws ProducerFencedException;  

//发送事务提交请求
void commitTransaction() throws ProducerFencedException;  

//发送事务回滚请求
void abortTransaction() throws ProducerFencedException;

}
```


上述方法通过transactionManager对象，向Transaction Coordinator发送事务请求，并处理响应结果，主要包含以下几类请求：

1. FindCoordinatorRequest：寻找Transaction Coordinator服务地址，用于sender线程发送事务请求前，或者其他响应结果返回coordinator错误等信息时调用。
2. InitProducerIdRequest：事务初始化请求
3. TxnOffsetCommitRequest：事务消费offset提交请求
4. AddPartitionsToTxnRequest：事务分区上传请求
5. EndTxnRequest：事务提交/回滚请i去


上述几类请求，都使用抽象类TxnRequestHandler进行包装，并处理来自Transaction Coordinator的响应结果。队列请求由Sender线程进行异步发送。

```java
//向broker transaction coordinator发送请求的优先级队列  
private final PriorityQueue&lt;TxnRequestHandler&gt; pendingRequests;
```


```java
abstract class TxnRequestHandler implements RequestCompletionHandler {  
    protected final TransactionalRequestResult result;  
    private boolean isRetry = false;

abstract void handleResponse(AbstractResponse responseBody);
}
```


![事务请求handler](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250402133808.png)


## 3.2. initializeTransactions

initializeTransactions方法提供给Producer初始化事务：

```java
synchronized TransactionalRequestResult initializeTransactions(ProducerIdAndEpoch producerIdAndEpoch) {  
    maybeFailWithError();  
  
    //1.1 检查ProducerIdAndEpoch是否为空  
    boolean isEpochBump = producerIdAndEpoch != ProducerIdAndEpoch.NONE;  
  
    //通过handleCachedTransactionRequestResult方法处理事务请求结果  
    return handleCachedTransactionRequestResult(() -&gt; {  
        // If this is an epoch bump, we will transition the state as part of handling the EndTxnRequest  
        if (!isEpochBump) {  
            //1.2 初始化事务状态为INITIALIZING  
            transitionTo(State.INITIALIZING);  
            log.info(&#34;Invoking InitProducerId for the first time in order to acquire a producer ID&#34;);  
        } else {  
            log.info(&#34;Invoking InitProducerId with current producer ID and epoch {} in order to bump the epoch&#34;, producerIdAndEpoch);  
        }  
        //1.3 向broker端发送InitProducerIdRequest请求  
        InitProducerIdRequestData requestData = new InitProducerIdRequestData()  
                .setTransactionalId(transactionalId)  
                .setTransactionTimeoutMs(transactionTimeoutMs)  
                .setProducerId(producerIdAndEpoch.producerId)  
                .setProducerEpoch(producerIdAndEpoch.epoch);  
        InitProducerIdHandler handler = new InitProducerIdHandler(new InitProducerIdRequest.Builder(requestData),  
                isEpochBump);  
        enqueueRequest(handler);  
        return handler.result;  
    }, State.INITIALIZING, &#34;initTransactions&#34;);  
}
```


InitProducerIdHandler对响应结果的处理较为简单：
1. 如果请求成功，读取ProducerIdAndEpoch，并设置事务状态为READY  
2. 如果是transaction coordinator类错误，则发送则发送FindCoordinatorRequest后重新发送InitProducerIdRequest
3. 如果响应中的错误是RetriableException，则重新发送InitProducerIdRequest  
4. 其他类型错误抛出异常

```java
public void handleResponse(AbstractResponse response) {  
    InitProducerIdResponse initProducerIdResponse = (InitProducerIdResponse) response;  
    Errors error = initProducerIdResponse.error();  
  
    //1.1 如果请求成功，读取ProducerIdAndEpoch，并设置事务状态为READY  
    if (error == Errors.NONE) {  
        ProducerIdAndEpoch producerIdAndEpoch = new ProducerIdAndEpoch(initProducerIdResponse.data().producerId(),  
                initProducerIdResponse.data().producerEpoch());  
        setProducerIdAndEpoch(producerIdAndEpoch);  
        transitionTo(State.READY);  
        lastError = null;  
        if (this.isEpochBump) {  
            resetSequenceNumbers();  
        }  
        result.done();  
    } else if (error == Errors.NOT_COORDINATOR || error == Errors.COORDINATOR_NOT_AVAILABLE) {  
        //1.2 如果响应中的错误是NOT_COORDINATOR或COORDINATOR_NOT_AVAILABLE，则发送FindCoordinatorRequest后重新发送InitProducerIdRequest  
        lookupCoordinator(FindCoordinatorRequest.CoordinatorType.TRANSACTION, transactionalId);  
        reenqueue();  
    } else if (error.exception() instanceof RetriableException) {  
        //1.3 如果响应中的错误是RetriableException，则重新发送InitProducerIdRequest  
        reenqueue();  
    } else if (error == Errors.TRANSACTIONAL_ID_AUTHORIZATION_FAILED ||  
            error == Errors.CLUSTER_AUTHORIZATION_FAILED) {  
        //1.4 其它类型的错误，抛出异常  
        log.info(&#34;Abortable authorization error: {}.  Transition the producer state to {}&#34;, error.message(), State.ABORTABLE_ERROR);  
        lastError = error.exception();  
        abortableError(error.exception());  
    } else if (error == Errors.INVALID_PRODUCER_EPOCH || error == Errors.PRODUCER_FENCED) {  
        // We could still receive INVALID_PRODUCER_EPOCH from old versioned transaction coordinator,  
        // just treat it the same as PRODUCE_FENCED.        fatalError(Errors.PRODUCER_FENCED.exception());  
    } else if (error == Errors.TRANSACTION_ABORTABLE) {  
        abortableError(error.exception());  
    } else {  
        fatalError(new KafkaException(&#34;Unexpected error in InitProducerIdResponse; &#34; &#43; error.message()));  
    }  
}
```


## 3.3. beginTransaction

beginTransaction方法用于开启事务，实现逻辑较为简单：将事务状态置为IN_TRANSACTION。

```java
public synchronized void beginTransaction() {  
    // 确保transactionalId事务id不为空  
    ensureTransactional();  
    throwIfPendingState(&#34;beginTransaction&#34;);  
    maybeFailWithError();  
  
    //状态置为IN_TRANSACTION  
    transitionTo(State.IN_TRANSACTION);  
}
```


## 3.4. addPartitionsToTransactionHandler


在消息发送流程org.apache.kafka.clients.producer.KafkaProducer#doSend中，开启了事务的情况下，会将topicPartition存储在transactionManager的newPartitionsInTransaction中：

```java
private Future&lt;RecordMetadata&gt; doSend(ProducerRecord&lt;K, V&gt; record, Callback callback) {
	if (transactionManager != null) {  
	    transactionManager.maybeAddPartition(appendCallbacks.topicPartition());  
	}
}

public synchronized void maybeAddPartition(TopicPartition topicPartition) {  
    maybeFailWithError();  
    throwIfPendingState(&#34;send&#34;);  
  
    if (isTransactional()) {  
        if (!hasProducerId()) {  
            throw new IllegalStateException(&#34;Cannot add partition &#34; &#43; topicPartition &#43;  
                &#34; to transaction before completing a call to initTransactions&#34;);  
        } else if (currentState != State.IN_TRANSACTION) {  
            throw new IllegalStateException(&#34;Cannot add partition &#34; &#43; topicPartition &#43;  
                &#34; to transaction while in state  &#34; &#43; currentState);  
        } else if (isPartitionAdded(topicPartition) || isPartitionPendingAdd(topicPartition)) {  
            return;  
        } else {  
            log.debug(&#34;Begin adding new partition {} to transaction&#34;, topicPartition);  
            txnPartitionMap.getOrCreate(topicPartition);  
            newPartitionsInTransaction.add(topicPartition);  
        }  
    }  
}
```


TransactionManager提供了以下容器用于存储各种状态的TopicPartition：

```java
//待封装成Request的TopicPartition
private final Set&lt;TopicPartition&gt; newPartitionsInTransaction;  
//已封装成Request的TopicPartition，待发送到broker
private final Set&lt;TopicPartition&gt; pendingPartitionsInTransaction; 
////已经上传的TopicPartition
private final Set&lt;TopicPartition&gt; partitionsInTransaction;
```


addPartitionsToTransactionHandler方法会将newPartitionsInTransaction封装到AddPartitionsToTxnRequest中：

```java
private TxnRequestHandler addPartitionsToTransactionHandler() {  
    pendingPartitionsInTransaction.addAll(newPartitionsInTransaction);  
    newPartitionsInTransaction.clear();  
    AddPartitionsToTxnRequest.Builder builder =  
        AddPartitionsToTxnRequest.Builder.forClient(transactionalId,  
            producerIdAndEpoch.producerId,  
            producerIdAndEpoch.epoch,  
            new ArrayList&lt;&gt;(pendingPartitionsInTransaction));  
    return new AddPartitionsToTxnHandler(builder);  
}
```


AddPartitionsToTxnRequest请求的发送时机有以下两种情况：
1. 提交/结束事务时，检查newPartitionsInTransaction是否为空
2. Sender线程发送事务请求时，nextRequest方法中会检查是否需要发送AddPartitionsToTxnRequest


AddPartitionsToTxnHandler中对于响应的处理逻辑较为简单，除去异常处理逻辑后，主要是处理partition容器的相关逻辑：

```java
public void handleResponse(AbstractResponse response) {  
    AddPartitionsToTxnResponse addPartitionsToTxnResponse = (AddPartitionsToTxnResponse) response;  
    Map&lt;TopicPartition, Errors&gt; errors = addPartitionsToTxnResponse.errors().get(AddPartitionsToTxnResponse.V3_AND_BELOW_TXN_ID);  
    boolean hasPartitionErrors = false;  
    Set&lt;String&gt; unauthorizedTopics = new HashSet&lt;&gt;();  
    retryBackoffMs = TransactionManager.this.retryBackoffMs;  

    Set&lt;TopicPartition&gt; partitions = errors.keySet();  
    pendingPartitionsInTransaction.removeAll(partitions);  
  
    if (!unauthorizedTopics.isEmpty()) {  
        abortableError(new TopicAuthorizationException(unauthorizedTopics));  
    } else if (hasPartitionErrors) {  
        abortableError(new KafkaException(&#34;Could not add partitions to transaction due to errors: &#34; &#43; errors));  
    } else {  
        log.debug(&#34;Successfully added partitions {} to transaction&#34;, partitions);  
        partitionsInTransaction.addAll(partitions);  
        transactionStarted = true;  
        result.done();  
    }  
}
```


## 3.5. sendOffsetsToTransaction

sendOffsetsToTransaction方法允许在 Kafka 事务中提交消费者的偏移量，确保消息处理和偏移量提交是一个原子操作。发送AddOffsetsToTxnRequest：

```java
public synchronized TransactionalRequestResult sendOffsetsToTransaction(final Map&lt;TopicPartition, OffsetAndMetadata&gt; offsets,  
                                                                        final ConsumerGroupMetadata groupMetadata) {  
    ensureTransactional();  
    throwIfPendingState(&#34;sendOffsetsToTransaction&#34;);  
    maybeFailWithError();  
  
    //检查是否为IN_TRANSACTION状态  
    if (currentState != State.IN_TRANSACTION) {  
        throw new IllegalStateException(&#34;Cannot send offsets if a transaction is not in progress &#34; &#43;  
            &#34;(currentState= &#34; &#43; currentState &#43; &#34;)&#34;);  
    }  
  
    //发送AddOffsetsToTxnRequest到broker端  
    log.debug(&#34;Begin adding offsets {} for consumer group {} to transaction&#34;, offsets, groupMetadata);  
    AddOffsetsToTxnRequest.Builder builder = new AddOffsetsToTxnRequest.Builder(  
        new AddOffsetsToTxnRequestData()  
            .setTransactionalId(transactionalId)  
            .setProducerId(producerIdAndEpoch.producerId)  
            .setProducerEpoch(producerIdAndEpoch.epoch)  
            .setGroupId(groupMetadata.groupId())  
    );  
    AddOffsetsToTxnHandler handler = new AddOffsetsToTxnHandler(builder, offsets, groupMetadata);  
  
    enqueueRequest(handler);  
    return handler.result;  
}
```


AddOffsetsToTxnHandler响应的处理逻辑很简单：如果请求成功，发送TxnOffsetCommitRequest给Group Coordinator。

```java
public void handleResponse(AbstractResponse response) {  
    AddOffsetsToTxnResponse addOffsetsToTxnResponse = (AddOffsetsToTxnResponse) response;  
    Errors error = Errors.forCode(addOffsetsToTxnResponse.data().errorCode());  
  
    if (error == Errors.NONE) {  
        //1.1 请求成功的情况  
        log.debug(&#34;Successfully added partition for consumer group {} to transaction&#34;, builder.data.groupId());  
  
        //1.2 成功后会发送TxnOffsetCommitRequest  
        // note the result is not completed until the TxnOffsetCommit returns
        pendingRequests.add(txnOffsetCommitHandler(result, offsets, groupMetadata));  
  
        transactionStarted = true;  
    }
}
```


## 3.6. beginCommit &amp;&amp; beginAbort


事务提交或回滚逻辑都是通过beginCompletingTransaction方法完成，区别在于事务状态的修改，beginCompletingTransaction方法会提前检查最近一次请求error是否为InvalidPidMappingException，若是则初始化事务状态，其他情况发送EndTxnRequest：

```java
private TransactionalRequestResult beginCompletingTransaction(TransactionResult transactionResult) {  
  
    //1.1 检查是否需要发送AddPartitionsToTxnRequest  
    if (!newPartitionsInTransaction.isEmpty())  
        enqueueRequest(addPartitionsToTransactionHandler());  
  
    // If the error is an INVALID_PRODUCER_ID_MAPPING error, the server will not accept an EndTxnRequest, so skip  
    // directly to InitProducerId. Otherwise, we must first abort the transaction, because the producer will be    // fenced if we directly call InitProducerId.  
    //1.2 如果是INVALID_PRODUCER_ID_MAPPING，broker不会接受EndTxnRequest请求，直接转为初始化initializeTransactions   
     if (!(lastError instanceof InvalidPidMappingException)) {  
        //1.3 向broker端发送EndTxnRequest请求  
        EndTxnRequest.Builder builder = new EndTxnRequest.Builder(  
                new EndTxnRequestData()  
                        .setTransactionalId(transactionalId)  
                        .setProducerId(producerIdAndEpoch.producerId)  
                        .setProducerEpoch(producerIdAndEpoch.epoch)  
                        .setCommitted(transactionResult.id));  
  
        EndTxnHandler handler = new EndTxnHandler(builder);  
        enqueueRequest(handler);  
        if (!epochBumpRequired) {  
            return handler.result;  
        }  
    }  
  
    return initializeTransactions(this.producerIdAndEpoch);  
}
```


EndTxnHandler中，会将TransactionManager状态改为READY。

```java
public void handleResponse(AbstractResponse response) {  
    EndTxnResponse endTxnResponse = (EndTxnResponse) response;  
    Errors error = endTxnResponse.error();  
  
    if (error == Errors.NONE) {  
        //1.1 请求成功，更新事务状态为READY  
        completeTransaction();  
        result.done();  
    } 
}
```


# 4. Broker端实现
 

## 4.1. TransactionMetadata

TransactionMetadata是一次事务的元数据，包含以下字段：

- transactionalId：事务唯一标识
- producerId：生产者id
- txnTimeoutMs：事务超时时间
- TransactionState：事务状态
- topicPartitions：本次事务涉及的TopicPartition

```scala
private[transaction] class TransactionMetadata(val transactionalId: String,  
                                               var producerId: Long,  
                                               var lastProducerId: Long,  
                                               var producerEpoch: Short,  
                                               var lastProducerEpoch: Short,  
                                               var txnTimeoutMs: Int,  
                                               var state: TransactionState,  
                                               val topicPartitions: mutable.Set[TopicPartition],  
                                               @volatile var txnStartTimestamp: Long = -1,  
                                               @volatile var txnLastUpdateTimestamp: Long) extends Logging {  
  
  // pending state is used to indicate the state that this transaction is going to  
  // transit to, and for blocking future attempts to transit it again if it is not legal;  // initialized as the same as the current state  
  var pendingState: Option[TransactionState] = None
  }
```


## 4.2. TransactionStateManager


负责管理：
1. 事务日志
2. 事务metadata
3. 事务过期逻辑



## 4.3. handleFindCoordinator

FindCoordinatorRequest是client端用于寻找Coordinator的请求，在broker端由kafka.server.KafkaApis#getCoordinator方法用于处理FindCoordinator请求（包含Group和Transaction两种类型），去除掉了参数校验的逻辑：

```scala
private def getCoordinator(request: RequestChannel.Request, keyType: Byte, key: String): (Errors, Node) = {  
  else {  
    val (partition, internalTopicName) = CoordinatorType.forId(keyType) match {  
      case CoordinatorType.GROUP =&gt;  
        (groupCoordinator.partitionFor(key), GROUP_METADATA_TOPIC_NAME)  
  
      case CoordinatorType.TRANSACTION =&gt;  
        (txnCoordinator.partitionFor(key), TRANSACTION_STATE_TOPIC_NAME)  
  
      case CoordinatorType.SHARE =&gt;  
        // When share coordinator support is implemented in KIP-932, a proper check will go here  
        return (Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)  
    }  
  
    val topicMetadata = metadataCache.getTopicMetadata(Set(internalTopicName), request.context.listenerName)  
  
    if (topicMetadata.headOption.isEmpty) {  
      val controllerMutationQuota = quotas.controllerMutation.newPermissiveQuotaFor(request)  
      autoTopicCreationManager.createTopics(Seq(internalTopicName).toSet, controllerMutationQuota, None)  
      (Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)  
    } else {  
      if (topicMetadata.head.errorCode != Errors.NONE.code) {  
        (Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)  
      } else {  
        val coordinatorEndpoint = topicMetadata.head.partitions.asScala  
          .find(_.partitionIndex == partition)  
          .filter(_.leaderId != MetadataResponse.NO_LEADER_ID)  
          .flatMap(metadata =&gt; metadataCache.  
              getAliveBrokerNode(metadata.leaderId, request.context.listenerName))  
  
        coordinatorEndpoint match {  
          case Some(endpoint) =&gt;  
            (Errors.NONE, endpoint)  
          case _ =&gt;  
            (Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)  
        }  
      }  
    }  
  }  
}
```


事务Coordinator定位逻辑通过传入transactionalId的hashcode对配置项`transaction.state.log.num.partitions`取模：

```scala
def partitionFor(transactionalId: String): Int = Utils.abs(transactionalId.hashCode) % transactionTopicPartitionCount
```

该配置默认值为50：

```java
public static final String TRANSACTIONS_TOPIC_PARTITIONS_CONFIG = &#34;transaction.state.log.num.partitions&#34;;  
public static final int TRANSACTIONS_TOPIC_PARTITIONS_DEFAULT = 50;
```


## 4.4. handleInitProducerId

broker端处理InitProducerIdRequest的流程如下：
1. 检查传入的transactionalId是否为空，若为空直接生成producerId并返回
2. 如果不为空，检查是否存在该transactionalId对应的事务信息
3. 如果不存在事务信息，则生成新的producerId，并更新事务信息到transaction metadata
4. 调用prepareInitProducerIdTransit方法，初始化事务状态
5. 如果当前事务状态为PrepareEpochFence，说明事务已经被新的producer使用，返回error，并结束该事务。
6. 提交事务信息到事务日志

```scala
def handleInitProducerId(transactionalId: String,  
                         transactionTimeoutMs: Int,  
                         expectedProducerIdAndEpoch: Option[ProducerIdAndEpoch],  
                         responseCallback: InitProducerIdCallback,  
                         requestLocal: RequestLocal = RequestLocal.NoCaching): Unit = {  
  
  //1.1 检查传入的transactionalId是否为空  
  if (transactionalId == null) {  
    // if the transactional id is null, then always blindly accept the request  
    // and return a new producerId from the producerId manager  
    //1.2 生成新的producerId    producerIdManager.generateProducerId() match {  
      case Success(producerId) =&gt;  
        responseCallback(InitProducerIdResult(producerId, producerEpoch = 0, Errors.NONE))  
      case Failure(exception) =&gt;  
        responseCallback(initTransactionError(Errors.forException(exception)))  
    }  
  } else if (transactionalId.isEmpty) {  
    // if transactional id is empty then return error as invalid request. This is  
    // to make TransactionCoordinator&#39;s behavior consistent with producer client    responseCallback(initTransactionError(Errors.INVALID_REQUEST))  
  } else if (!txnManager.validateTransactionTimeoutMs(transactionTimeoutMs)) {  
    // check transactionTimeoutMs is not larger than the broker configured maximum allowed value  
    responseCallback(initTransactionError(Errors.INVALID_TRANSACTION_TIMEOUT))  
  } else {  
    //2.1 检查是否存在该transactionalId对应的事务信息  
    val coordinatorEpochAndMetadata = txnManager.getTransactionState(transactionalId).flatMap {  
      case None =&gt;  
        //2.2 如果不存在事务信息，生成新的producerId，并更新事务信息到metadata  
        producerIdManager.generateProducerId() match {  
          case Success(producerId) =&gt;  
            val createdMetadata = new TransactionMetadata(transactionalId = transactionalId,  
              producerId = producerId,  
              lastProducerId = RecordBatch.NO_PRODUCER_ID,  
              producerEpoch = RecordBatch.NO_PRODUCER_EPOCH,  
              lastProducerEpoch = RecordBatch.NO_PRODUCER_EPOCH,  
              txnTimeoutMs = transactionTimeoutMs,  
              state = Empty,  
              topicPartitions = collection.mutable.Set.empty[TopicPartition],  
              txnLastUpdateTimestamp = time.milliseconds())  
            txnManager.putTransactionStateIfNotExists(createdMetadata)  
  
          case Failure(exception) =&gt;  
            Left(Errors.forException(exception))  
        }  
  
      case Some(epochAndTxnMetadata) =&gt; Right(epochAndTxnMetadata)  
    }  
  
    val result: ApiResult[(Int, TxnTransitMetadata)] = coordinatorEpochAndMetadata.flatMap {  
      existingEpochAndMetadata =&gt;  
        val coordinatorEpoch = existingEpochAndMetadata.coordinatorEpoch  
        val txnMetadata = existingEpochAndMetadata.transactionMetadata  
  
        //3.1 调用prepareInitProducerIdTransit方法，初始化事务  
        txnMetadata.inLock {  
          prepareInitProducerIdTransit(transactionalId, transactionTimeoutMs, coordinatorEpoch, txnMetadata,  
            expectedProducerIdAndEpoch)  
        }  
    }  
  
    result match {  
      case Left(error) =&gt;  
        responseCallback(initTransactionError(error))  
  
      case Right((coordinatorEpoch, newMetadata)) =&gt;  
        //4.1 如果当前事务状态为PrepareEpochFence，说明事务已经被新的producer使用，直接返回error  
        if (newMetadata.txnState == PrepareEpochFence) {  
          // abort the ongoing transaction and then return CONCURRENT_TRANSACTIONS to let client wait and retry  
          def sendRetriableErrorCallback(error: Errors): Unit = {  
            if (error != Errors.NONE) {  
              responseCallback(initTransactionError(error))  
            } else {  
              responseCallback(initTransactionError(Errors.CONCURRENT_TRANSACTIONS))  
            }  
          }  
  
          //4.2 结束当前事务  
          endTransaction(transactionalId,  
            newMetadata.producerId,  
            newMetadata.producerEpoch,  
            TransactionResult.ABORT,  
            isFromClient = false,  
            sendRetriableErrorCallback,  
            requestLocal)  
        } else {  
          def sendPidResponseCallback(error: Errors): Unit = {  
            if (error == Errors.NONE) {  
              info(s&#34;Initialized transactionalId $transactionalId with producerId ${newMetadata.producerId} and producer &#34; &#43;  
                s&#34;epoch ${newMetadata.producerEpoch} on partition &#34; &#43;  
                s&#34;${Topic.TRANSACTION_STATE_TOPIC_NAME}-${txnManager.partitionFor(transactionalId)}&#34;)  
              responseCallback(initTransactionMetadata(newMetadata))  
            } else {  
              info(s&#34;Returning $error error code to client for $transactionalId&#39;s InitProducerId request&#34;)  
              responseCallback(initTransactionError(error))  
            }  
          }  
  
          //5.1 提交事务信息到事务日志  
          txnManager.appendTransactionToLog(transactionalId, coordinatorEpoch, newMetadata,  
            sendPidResponseCallback, requestLocal = requestLocal)  
        }  
    }  
  }  
}
```


## 4.5. handleAddPartitionsToTransaction

AddPartitionsToTxnRequest和AddOffsetsToTxnRequest请求都是由handleAddPartitionsToTransaction方法进行处理，区别在于后者的TopicPartition对象为`__consumer_offsets`，而前者是具体的业务topic。

它的处理逻辑较为简单：

1.  将TopicPartition写入内存metadata中
2. 将事务metadata写入事务日志持久化

```scala
def handleAddPartitionsToTransaction(transactionalId: String,  
                                     producerId: Long,  
                                     producerEpoch: Short,  
                                     partitions: collection.Set[TopicPartition],  
                                     responseCallback: AddPartitionsCallback,  
                                     requestLocal: RequestLocal = RequestLocal.NoCaching): Unit = {  
  if (transactionalId == null || transactionalId.isEmpty) {  
    debug(s&#34;Returning ${Errors.INVALID_REQUEST} error code to client for $transactionalId&#39;s AddPartitions request&#34;)  
    responseCallback(Errors.INVALID_REQUEST)  
  } else {  

    val result: ApiResult[(Int, TxnTransitMetadata)] = txnManager.getTransactionState(transactionalId).flatMap {  
      case None =&gt; Left(Errors.INVALID_PRODUCER_ID_MAPPING)  
  
      case Some(epochAndMetadata) =&gt;  
        val coordinatorEpoch = epochAndMetadata.coordinatorEpoch  
        val txnMetadata = epochAndMetadata.transactionMetadata  
  
        // generate the new transaction metadata with added partitions  
        txnMetadata.inLock {  
          else {  
            Right(coordinatorEpoch, txnMetadata.prepareAddPartitions(partitions.toSet, time.milliseconds()))  
          }  
        }  
    }  
  
    result match {  
      case Left(err) =&gt;  
        debug(s&#34;Returning $err error code to client for $transactionalId&#39;s AddPartitions request&#34;)  
        responseCallback(err)  
	  ////2.1 将事务信息写入事务日志中
      case Right((coordinatorEpoch, newMetadata)) =&gt;  
        txnManager.appendTransactionToLog(transactionalId, coordinatorEpoch, newMetadata,  
          responseCallback, requestLocal = requestLocal)  
    }  
  }  
}
```


## 4.6. handleTxnCommitOffsets

TxnOffsetCommitRequest的请求处理逻辑由Group Coordinator进行处理，


```scala
def handleTxnCommitOffsets(groupId: String,  
                           transactionalId: String,  
                           producerId: Long,  
                           producerEpoch: Short,  
                           memberId: String,  
                           groupInstanceId: Option[String],  
                           generationId: Int,  
                           offsetMetadata: immutable.Map[TopicIdPartition, OffsetAndMetadata],  
                           responseCallback: immutable.Map[TopicIdPartition, Errors] =&gt; Unit,  
                           requestLocal: RequestLocal = RequestLocal.NoCaching,  
                           apiVersion: Short): Unit = {  
  validateGroupStatus(groupId, ApiKeys.TXN_OFFSET_COMMIT) match {  
  
    //1.1 验证消费者组的状态  
    case Some(error) =&gt; responseCallback(offsetMetadata.map { case (k, _) =&gt; k -&gt; error })  
    case None =&gt;  
      //1.2 获取消费者组信息  
      val group = groupManager.getGroup(groupId).getOrElse {  
        groupManager.addGroup(new GroupMetadata(groupId, Empty, time))  
      }  
  
      //1.3 获取偏移主题分区信息  
      val offsetTopicPartition = new TopicPartition(Topic.GROUP_METADATA_TOPIC_NAME, partitionFor(group.groupId))  
  
      def postVerificationCallback(  
        newRequestLocal: RequestLocal,  
        errorAndGuard: (Errors, VerificationGuard)  
      ): Unit = {  
        val (error, verificationGuard) = errorAndGuard  
        if (error != Errors.NONE) {  
          val finalError = GroupMetadataManager.maybeConvertOffsetCommitError(error)  
          responseCallback(offsetMetadata.map { case (k, _) =&gt; k -&gt; finalError })  
        } else {  
  
          //2.1 存储事务提交偏移  
          doTxnCommitOffsets(group, memberId, groupInstanceId, generationId, producerId, producerEpoch,  
            offsetTopicPartition, offsetMetadata, newRequestLocal, responseCallback, Some(verificationGuard))  
        }  
      }  
  
      val transactionSupportedOperation = if (apiVersion &gt;= 4) genericError else defaultError  
  
      //3.1 检查topicPartition是否支持事务性写入  
      groupManager.replicaManager.maybeStartTransactionVerificationForPartition(  
        topicPartition = offsetTopicPartition,  
        transactionalId,  
        producerId,  
        producerEpoch,  
        RecordBatch.NO_SEQUENCE,  
        // Wrap the callback to be handled on an arbitrary request handler thread  
        // when transaction verification is complete. The request local passed in        // is only used when the callback is executed immediately.        
        KafkaRequestHandler.wrapAsyncCallback(  
          postVerificationCallback,  
          requestLocal  
        ),  
        transactionSupportedOperation  
      )  
  }  
}
```


在实际写入消费记录时，会检查是否为事务提交：

```scala
if (isTxnOffsetCommit) {  
  addProducerGroup(producerId, group.groupId)  
  group.prepareTxnOffsetCommit(producerId, filteredOffsetMetadata)  
} else {  
  group.prepareOffsetCommit(filteredOffsetMetadata)  
}
```

提交的消费位移记录会暂时保存在map中，等执行结束/回滚事务时再执行相应操作：

```scala
private val pendingTransactionalOffsetCommits = new mutable.HashMap[Long, mutable.Map[TopicPartition, CommitRecordMetadataAndOffset]]() 

def prepareTxnOffsetCommit(producerId: Long, offsets: Map[TopicIdPartition, OffsetAndMetadata]): Unit = {  
  trace(s&#34;TxnOffsetCommit for producer $producerId and group $groupId with offsets $offsets is pending&#34;)  
  receivedTransactionalOffsetCommits = true  
  val producerOffsets = pendingTransactionalOffsetCommits.getOrElseUpdate(producerId,  
    mutable.Map.empty[TopicPartition, CommitRecordMetadataAndOffset])  
  
  offsets.forKeyValue { (topicIdPartition, offsetAndMetadata) =&gt;  
    producerOffsets.put(topicIdPartition.topicPartition, CommitRecordMetadataAndOffset(None, offsetAndMetadata))  
  }  
}
```


## 4.7. handleEndTransaction

EndTxnRequest请求由handleEndTransaction方法进行处理，主要分为以下步骤：
1. 更新事务元数据中的事务状态
2. 向TopicPartition的leader endpoint发送WriteTxnMarkerRequest
3. 将事务结果写入事务日志中

```scala
def handleEndTransaction(transactionalId: String,  
                         producerId: Long,  
                         producerEpoch: Short,  
                         txnMarkerResult: TransactionResult,  
                         responseCallback: EndTxnCallback,  
                         requestLocal: RequestLocal = RequestLocal.NoCaching): Unit = {  
  endTransaction(transactionalId,  
    producerId,  
    producerEpoch,  
    txnMarkerResult,  
    isFromClient = true,  
    responseCallback,  
    requestLocal)  
}
```



## 4.8. scheduleHandleTxnCompletion

WriteTxnMarkerRequest请求由leader partition所在的broker处理，异步执行更新group metadata中的消费进度kafka.coordinator.group.GroupMetadata#offsets。

```scala
def scheduleHandleTxnCompletion(producerId: Long, completedPartitions: Set[Int], isCommit: Boolean): CompletableFuture[Void] = {  
  val future = new CompletableFuture[Void]()  
  scheduler.scheduleOnce(s&#34;handleTxnCompletion-$producerId&#34;, () =&gt; {  
    try {  
      handleTxnCompletion(producerId, completedPartitions, isCommit)  
      future.complete(null)  
    } catch {  
      case e: Throwable =&gt; future.completeExceptionally(e)  
    }  
  })  
  future  
}
```


```scala
private[group] def handleTxnCompletion(producerId: Long, completedPartitions: Set[Int], isCommit: Boolean): Unit = {  
  val pendingGroups = groupsBelongingToPartitions(producerId, completedPartitions)  
  pendingGroups.foreach { groupId =&gt;  
    getGroup(groupId) match {  
      case Some(group) =&gt; group.inLock {  
        if (!group.is(Dead)) {  
          group.completePendingTxnOffsetCommit(producerId, isCommit)  
          removeProducerGroup(producerId, groupId)  
        }  
      }  
      case _ =&gt;  
        info(s&#34;Group $groupId has moved away from $brokerId after transaction marker was written but before the &#34; &#43;  
          s&#34;cache was updated. The cache on the new group owner will be updated instead.&#34;)  
    }  
  }  
}
```



# 5. Reference

[深入 Kafka Core 的设计（事务篇） – K&#39;s Blog](https://nlogn.art/dive-into-kafka-design-part-3.html)

[Kafka Transactional Support: How It Enables Exactly-Once Semantics](https://developer.confluent.io/courses/architecture/transactions/)

[Transactions in Apache Kafka \| Confluent](https://www.confluent.io/blog/transactions-apache-kafka/)

[Kafka Exactly-Once 之事务性实现 \| Matt&#39;s Blog](https://matt33.com/2018/11/04/kafka-transaction/)

[Kafka 事务实现原理 \| 学习笔记](https://zhmin.github.io/posts/kafka-transaction/)

[KIP-98 - Exactly Once Delivery and Transactional Messaging - Apache Kafka - Apache Software Foundation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98&#43;-&#43;Exactly&#43;Once&#43;Delivery&#43;and&#43;Transactional&#43;Messaging)


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-7-transaction-design/  

