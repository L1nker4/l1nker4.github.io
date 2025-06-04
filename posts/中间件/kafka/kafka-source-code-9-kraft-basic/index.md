# Kafka源码分析(九)- KRaft架构简析




# 1. Intro


Kafka属于单Master多Worker结构，Zookeeper主要提供了metadata存储、分布式同步、集群Leader选举等功能。

至于为何当下Kafka抛弃Zookeeper转而选择自建Raft代替，也是一个老生常谈的问题：
1. Zookeeper作为单独的分布式系统，加大了Kafka集群的部署、运维成本。
2. Zookeeper存在性能瓶颈，无法支撑更大的集群规模，而自建KRaft支持更大数据量级的分区元数据管理。

![image.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250408102752344.png)


下文会对Kafka提出的KRaft架构做简要分析。


# 2. Raft算法回顾


Raft算法是一种用于保证分布式集群中数据一致性算法，在[Raft论文](https://raft.github.io/raft.pdf) 中也明确提出：正是因为Paxos算法较为复杂，并且不易于应用到工业界，所以构建了更易于理解的Raft算法来解决分布式系统中的一致性问题。


Raft集群中只有三种角色：领导者、候选者、跟随者。Raft算法中每一段任期只有一个领导者节点，每一段任期从一次选举开始，一个或多个候选者参加选举，赢得选举将在接下来的任期充当领导者。跟随者只响应其他服务器的请求。下图为三种状态的转换关系：

![image.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250316162555.png)




# 3. 架构简析

在KRaft架构之前，整个Kafka集群通过Zookeeper来实现元数据存储、选举等目的，通过Zookeeper来选举是通过创建临时节点来实现的，架构实现可以参考下图（[confluent](https://developer.confluent.io/courses/architecture/control-plane/)）：
![image.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250312134915.png)



KRaft模式中，会选取其中部分Broker作为Controller来完成metadata存储和集群管理，但某个时间段只能有一个active Controller，它负责处理整个集群中的RPC请求，其他用作热备的Controller会从active节点同步数据。

![image.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250312135841.png)


需要注意的是，在日志复制的实现上，KRaft并没有完全采用原生Raft的方式：Leader节点主动往Follower节点推送，而是让Follower主动从Leader执行Fetch。

![message-model](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250408102655411.png)

# 4. KRaft 基本结构


KRaft相关提案：

KRaft概述：[KIP-500](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A&#43;Replace&#43;ZooKeeper&#43;with&#43;a&#43;Self-Managed&#43;Metadata&#43;Quorum)
KRaft复制/quorum维护协议规范：[KIP-595](https://cwiki.apache.org/confluence/display/KAFKA/KIP-595%3A&#43;A&#43;Raft&#43;Protocol&#43;for&#43;the&#43;Metadata&#43;Quorum#KIP595:ARaftProtocolfortheMetadataQuorum-ParentKIP)
日志压缩：[KIP-630](https://cwiki.apache.org/confluence/display/KAFKA/KIP-630%3A&#43;Kafka&#43;Raft&#43;Snapshot)


KRaft具体实现位于`org.apache.kafka.raft`，snapshot被拆分到`org.apache.kafka.snapshot`中。

- KafkaRaftManager：KRaft管理与入口类，封装了dir manager、register listener、handleRequest等接口，内部实现通过调用KafkaRaftClient和KafkaRaftClientDriver完成。
- KafkaRaftClientDriver：Thread类，负责运行、管理KafkaRaftClient的生命周期。
- KafkaRaftClient：Kafka Raft协议的具体实现，实现RaftClient接口，选举策略采用原生Raft协议，日志复制是follower主动从leader拉取（符合Kafka语义）。


KRaft提供了Listener接口，提供了commit、snapshot等事件的回调接口。

```java
interface Listener&lt;T&gt; {  
    void handleCommit(BatchReader&lt;T&gt; reader);  
  
    void handleLoadSnapshot(SnapshotReader&lt;T&gt; reader);  
  
    default void handleLeaderChange(LeaderAndEpoch leader) {}  
  
    default void beginShutdown() {}  
}
```


RaftClient抽取的通用方法如下：
```java
void register(Listener&lt;T&gt; listener);  
  
void unregister(Listener&lt;T&gt; listener);
 
long prepareAppend(int epoch, List&lt;T&gt; records);  
  
void schedulePreparedAppend();  
  
CompletableFuture&lt;Void&gt; shutdown(int timeoutMs);  
  
void resign(int epoch);  
  
Optional&lt;SnapshotWriter&lt;T&gt;&gt; createSnapshot(OffsetAndEpoch snapshotId, long lastContainedLogTime);
```


## 4.1. listener注册逻辑

listener采用lazy-register方式，在消费RaftMessage的前置处理中，将一个时间段内的listener进行注册。

```java

//存储RaftMessage的队列
private final RaftMessageQueue messageQueue;

//listener上下文信息
private final Map&lt;Listener&lt;T&gt;, ListenerContext&gt; listenerContexts = new IdentityHashMap&lt;&gt;(); 

//待注册的listener信息
private final ConcurrentLinkedQueue&lt;Registration&lt;T&gt;&gt; pendingRegistrations = new ConcurrentLinkedQueue&lt;&gt;();
```


注册Listener会往pendingRegistrations追加，并唤醒messageQueue执行poll方法完成前置注册逻辑。register和unregister动作通过包装类Registration中的ops字段来区分。

```java
@Override  
public void register(Listener&lt;T&gt; listener) {  
    pendingRegistrations.add(Registration.register(listener));  
    wakeup();  
}

@Override  
public void unregister(Listener&lt;T&gt; listener) {  
    pendingRegistrations.add(Registration.unregister(listener));  
    // No need to wake up the polling thread. It is a removal so the updates can be  
    // delayed until the polling thread wakes up for other reasons.}

private void wakeup() {  
    messageQueue.wakeup();  
}

private void pollListeners() {  
    // Apply all of the pending registration  
    while (true) {  
        Registration&lt;T&gt; registration = pendingRegistrations.poll();  
        if (registration == null) {  
            break;  
        }  
  
        processRegistration(registration);  
    }
}

private void processRegistration(Registration&lt;T&gt; registration) {  
    Listener&lt;T&gt; listener = registration.listener();  
    Registration.Ops ops = registration.ops();  
  
    if (ops == Registration.Ops.REGISTER) {  
        if (listenerContexts.putIfAbsent(listener, new ListenerContext(listener)) != null) {  
            logger.error(&#34;Attempting to add a listener that already exists: {}&#34;, listenerName(listener));  
        } else {  
            logger.info(&#34;Registered the listener {}&#34;, listenerName(listener));  
        }
    } else {  
        if (listenerContexts.remove(listener) == null) {  
            logger.error(&#34;Attempting to remove a listener that doesn&#39;t exists: {}&#34;, listenerName(listener));  
        } else {  
            logger.info(&#34;Unregistered the listener {}&#34;, listenerName(listener));  
        }  
    }  
}

```


kafka.Kafka.buildServer在创建Server对象时，会根据配置，选择对应Sercer。
```scala
private def buildServer(props: Properties): Server = {  
  val config = KafkaConfig.fromProps(props, doLog = false)  
  if (config.requiresZookeeper) {  
    new KafkaServer(  
      config,  
      Time.SYSTEM,  
      threadNamePrefix = None,  
      enableForwarding = enableApiForwarding(config)  
    )  
  } else {  
    new KafkaRaftServer(  
      config,  
      Time.SYSTEM,  
    )  
  }  
}
```


## 4.2. KafkaRaftManager

KafkaRaftManager作为KRaft的入口类，从kafka.server.ControllerApis#handleRaftRequest中可以体现，涉及Raft相关的请求

![image.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250324132339.png)


```java
private def handleRaftRequest(request: RequestChannel.Request,  
                              buildResponse: ApiMessage =&gt; AbstractResponse): CompletableFuture[Unit] = {  
  val requestBody = request.body[AbstractRequest]  
  val future = raftManager.handleRequest(request.context, request.header, requestBody.data, time.milliseconds())  
  future.handle[Unit] { (responseData, exception) =&gt;  
    val response = if (exception != null) {  
      requestBody.getErrorResponse(exception)  
    } else {  
      buildResponse(responseData)  
    }  
    requestHelper.sendResponseExemptThrottle(request, response)  
  }  
}
```


在KafkaRaftManager中，也是直接将Request传递给Driver线程：

```scala
override def handleRequest(  
  context: RequestContext,  
  header: RequestHeader,  
  request: ApiMessage,  
  createdTimeMs: Long  
): CompletableFuture[ApiMessage] = {  
  clientDriver.handleRequest(context, header, request, createdTimeMs)  
}
```


其它代理方法：

```scala
override def register(  
  listener: RaftClient.Listener[T]  
): Unit = {  
  client.register(listener)  
}

override def leaderAndEpoch: LeaderAndEpoch = {  
  client.leaderAndEpoch  
}  
  
override def voterNode(id: Int, listener: ListenerName): Option[Node] = {  
  client.voterNode(id, listener).toScala  
}
```

## 4.3. KafkaRaftClientDriver


KafkaRaftClientDriver作为RaftClient处理线程，与KafkaRaftClient生命周期绑定，继承自ShutdownableThread类，doWrok逻辑中会一直调用`client.poll()`来处理inbound和outbound请求，有关poll细节在下文再做分析。

```java
private final KafkaRaftClient&lt;T&gt; client;

@Override  
public void doWork() {  
    try {  
        client.poll();  
    } catch (Throwable t) {  
        throw fatalFaultHandler.handleFault(&#34;Unexpected error in raft IO thread&#34;, t);  
    }  
}
```


shutdown逻辑中会首先shutdown RaftClient：

```java
@Override  
public boolean initiateShutdown() {  
    if (super.initiateShutdown()) {  
        client.shutdown(5000).whenComplete((na, exception) -&gt; {  
            if (exception != null) {  
                log.error(&#34;Graceful shutdown of RaftClient failed&#34;, exception);  
            } else {  
                log.info(&#34;Completed graceful shutdown of RaftClient&#34;);  
            }  
        });  
        return true;  
    } else {  
        return false;  
    }  
}  
  
@Override  
public void shutdown() throws InterruptedException {  
    try {  
        super.shutdown();  
    } finally {  
        client.close();  
    }  
}
```


## 4.4. KafkaRaftClient


KafkaRaftClient是Raft协议在Kafka中的具体实现，包含：leader选举、日志复制、日志snapshot等功能。


核心属性如下：

```java
//节点id  
private final OptionalInt nodeId;  
  
//节点目录id  
private final Uuid nodeDirectoryId;  
  
//  
private final AtomicReference&lt;GracefulShutdown&gt; shutdown = new AtomicReference&lt;&gt;();  
private final LogContext logContext;  
private final Logger logger;  
private final Time time;  
  
//fetch最大等待时间  
private final int fetchMaxWaitMs;  
private final boolean followersAlwaysFlush;  
  
//集群id  
private final String clusterId;  
  
//listener endpoint地址  
private final Endpoints localListeners;  
  
  
private final SupportedVersionRange localSupportedKRaftVersion;  
  
  
private final NetworkChannel channel;  
  
//本地日志对象  
private final ReplicatedLog log;  
private final Random random;  
  
//延迟任务  
private final FuturePurgatory&lt;Long&gt; appendPurgatory;  
private final FuturePurgatory&lt;Long&gt; fetchPurgatory;  
  
//log序列化工具
private final RecordSerde&lt;T&gt; serde;  
  
//内存池  
private final MemoryPool memoryPool;  
  
//raft事件消息队列  
private final RaftMessageQueue messageQueue;  
  
private final QuorumConfig quorumConfig;  
  
//raft snapshot清理  
private final RaftMetadataLogCleanerManager snapshotCleaner;  
  
//listeners  
private final Map&lt;Listener&lt;T&gt;, ListenerContext&gt; listenerContexts = new IdentityHashMap&lt;&gt;();  
  
//等待注册的listener队列  
private final ConcurrentLinkedQueue&lt;Registration&lt;T&gt;&gt; pendingRegistrations = new ConcurrentLinkedQueue&lt;&gt;();  
  
//kraft状态机  
private volatile KRaftControlRecordStateMachine partitionState;  
  
//kraft监控指标  
private volatile KafkaRaftMetrics kafkaRaftMetrics;  
  
//Quorum状态  
private volatile QuorumState quorum;  
  
//管理与其他controller的通信  
private volatile RequestManager requestManager;  
  
// Specialized handlers  
private volatile AddVoterHandler addVoterHandler;  
private volatile RemoveVoterHandler removeVoterHandler;  
private volatile UpdateVoterHandler updateVoterHandler;
```


通过KafkaRaftClientDriver线程驱动的事件逻辑：

```java
public void poll() {  
    if (!isInitialized()) {  
        throw new IllegalStateException(&#34;Replica needs to be initialized before polling&#34;);  
    }  
  
    long startPollTimeMs = time.milliseconds();  
    if (maybeCompleteShutdown(startPollTimeMs)) {  
        return;  
    }  
  
    //1.1 根据不同节点角色，执行对应逻辑  
    long pollStateTimeoutMs = pollCurrentState(startPollTimeMs);  
  
    //1.2 定期清理日志snapshot  
    long cleaningTimeoutMs = snapshotCleaner.maybeClean(startPollTimeMs);  
    long pollTimeoutMs = Math.min(pollStateTimeoutMs, cleaningTimeoutMs);  
  
    //1.3 更新指标  
    long startWaitTimeMs = time.milliseconds();  
    kafkaRaftMetrics.updatePollStart(startWaitTimeMs);  
  
    //1.4 获取inbound消息并处理  
    RaftMessage message = messageQueue.poll(pollTimeoutMs);  
  
    long endWaitTimeMs = time.milliseconds();  
    kafkaRaftMetrics.updatePollEnd(endWaitTimeMs);  
  
    if (message != null) {  
        handleInboundMessage(message, endWaitTimeMs);  
    }  
  
    //1.5 处理新注册的listener  
    pollListeners();  
}
```

## 4.5. KafkaRaftServer

`KafkaRaftServer`是Kraft模式下的Bootstrap启动类，一共有三个较为重要的对象，分别是：
- sharedServer：管理BrokerServer和ControllerServer共享的组件，例如：RaftManager、MetadataLoader等
- broker：broker对象，管理broker相关组件，例如：LogManager、GroupCoordinator等
- controller：controller对象，处理集群管理等事件

后两者通过具体配置`process.roles`进行选择性加载。

```scala
private val sharedServer = new SharedServer(  
  config,  
  metaPropsEnsemble,  
  time,  
  metrics,  
  CompletableFuture.completedFuture(QuorumConfig.parseVoterConnections(config.quorumVoters)),  
  QuorumConfig.parseBootstrapServers(config.quorumBootstrapServers),  
  new StandardFaultHandlerFactory(),  
)  
  
private val broker: Option[BrokerServer] = if (config.processRoles.contains(ProcessRole.BrokerRole)) {  
  Some(new BrokerServer(sharedServer))  
} else {  
  None  
}  
  
private val controller: Option[ControllerServer] = if (config.processRoles.contains(ProcessRole.ControllerRole)) {  
  Some(new ControllerServer(  
    sharedServer,  
    KafkaRaftServer.configSchema,  
    bootstrapMetadata,  
  ))  
} else {  
  None  
}
```



## 4.6. ControllerServer

ControllerServer作为KRaft模式下的controller Bootstrap类，内部属性如下：

```scala
var socketServer: SocketServer = _
var controller: Controller = _
var controllerApis: ControllerApis = _  
var controllerApisHandlerPool: KafkaRequestHandlerPool = _
val metadataPublishers: util.List[MetadataPublisher] = new util.ArrayList[MetadataPublisher]()
```

其中ControllerApis是所有Controller API的路由，负责调度各类Kafka control请求：

```scala
override def handle(request: RequestChannel.Request, requestLocal: RequestLocal): Unit = {  
  try {  
    val handlerFuture: CompletableFuture[Unit] = request.header.apiKey match {  
      case ApiKeys.FETCH =&gt; handleFetch(request)  
      case ApiKeys.FETCH_SNAPSHOT =&gt; handleFetchSnapshot(request)  
      case ApiKeys.CREATE_TOPICS =&gt; handleCreateTopics(request)  
      case ApiKeys.DELETE_TOPICS =&gt; handleDeleteTopics(request)  
      case ApiKeys.API_VERSIONS =&gt; handleApiVersionsRequest(request)
      }
   }
}      
```



## 4.7. QuorumController

Controller是对集群管理动作的抽象接口：

```java
public interface Controller extends AclMutator, AutoCloseable {  
    /**  
     * Change partition information. 
     * @param context       The controller request context.  
     * @param request       The AlterPartitionRequest data.  
     * @return              A future yielding the response.  
     */    CompletableFuture&lt;AlterPartitionResponseData&gt; alterPartition(  
        ControllerRequestContext context,  
        AlterPartitionRequestData request  
    );  
  
    /**  
     * Alter the user SCRAM credentials.        
     * @param context       The controller request context.  
     * @param request       The AlterUserScramCredentialsRequest data.  
     * @return              A future yielding the response.  
     */    CompletableFuture&lt;AlterUserScramCredentialsResponseData&gt; alterUserScramCredentials(  
        ControllerRequestContext context,  
        AlterUserScramCredentialsRequestData request  
    );
}
```

当前仅有一个实现类：org.apache.kafka.controller.QuorumController，QuorumController是对Quorum组节点的抽象，Quorum可以参考：[# Quorum (分布式系统))](https://zh.wikipedia.org/wiki/Quorum_(%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F))

QuorumController划分出多个manager进行功能划分：

```java
/**  
 * An object which stores the controller&#39;s view of the cluster. * This must be accessed only by the event queue thread. */
private final ClusterControlManager clusterControl;  
  
/**  
 * An object which stores the controller&#39;s view of the cluster features. * This must be accessed only by the event queue thread. */
private final FeatureControlManager featureControl;  
  
/**  
 * An object which stores the controller&#39;s view of the latest producer ID * that has been generated. This must be accessed only by the event queue thread. */
private final ProducerIdControlManager producerIdControlManager;  
  
/**  
 * An object which stores the controller&#39;s view of topics and partitions. * This must be accessed only by the event queue thread. */
private final ReplicationControlManager replicationControl;
```


这里以创建topic为例，来看下在KRaft架构下的处理流程。

```java
@Override  
public CompletableFuture&lt;CreateTopicsResponseData&gt; createTopics(  
    ControllerRequestContext context,  
    CreateTopicsRequestData request, Set&lt;String&gt; describable  
) {  
    if (request.topics().isEmpty()) {  
        return CompletableFuture.completedFuture(new CreateTopicsResponseData());  
    }  
    return appendWriteEvent(&#34;createTopics&#34;, context.deadlineNs(),  
        () -&gt; replicationControl.createTopics(context, request, describable));  
}


&lt;T&gt; CompletableFuture&lt;T&gt; appendWriteEvent(  
    String name,  
    OptionalLong deadlineNs,  
    ControllerWriteOperation&lt;T&gt; op,  
    EnumSet&lt;ControllerOperationFlag&gt; flags  
) {  
    ControllerWriteEvent&lt;T&gt; event = new ControllerWriteEvent&lt;&gt;(name, op, flags);  
    if (deadlineNs.isPresent()) {  
        queue.appendWithDeadline(deadlineNs.getAsLong(), event);  
    } else {  
        queue.append(event);  
    }  
    return event.future();  
}
```


`() -&gt; replicationControl.createTopics(context, request, describable)`是controller处理topic创建的逻辑，它会被封装成ControllerWriteEvent并写入到队列KafkaEventQueue中。


### 4.7.1. KafkaEventQueue

KafkaEventQueue是用于处理Controller事件的异步队列，实现自接口EventQueue，该接口封装了append、scheduleDeferred等方法。

```java
public interface EventQueue extends AutoCloseable {  
    interface Event {  
        void run() throws Exception;  
  
        default void handleException(Throwable e) {}  
    }

default void append(Event event) {  
    enqueue(EventInsertionType.APPEND, null, NoDeadlineFunction.INSTANCE, event);  
}

void enqueue(EventInsertionType insertionType,  
             String tag,  
             Function&lt;OptionalLong, OptionalLong&gt; deadlineNsCalculator,  
             Event event);

default void scheduleDeferred(String tag,  
                              Function&lt;OptionalLong, OptionalLong&gt; deadlineNsCalculator,  
                              Event event) {  
    enqueue(EventInsertionType.DEFERRED, tag, deadlineNsCalculator, event);
```


KafkaEventQueue通过EventHandler完成事件写入、处理。

```java
@Override  
public void enqueue(EventInsertionType insertionType,  
                    String tag,  
                    Function&lt;OptionalLong, OptionalLong&gt; deadlineNsCalculator,  
                    Event event) {  
    EventContext eventContext = new EventContext(event, insertionType, tag);  
    Exception e = eventHandler.enqueue(eventContext, deadlineNsCalculator);  
    if (e != null) {  
        eventContext.completeWithException(log, e);  
    }  
}


```

KafkaEventQueue内部字段如下，初始化时会启动处理线程。

```java
//事件处理器，封装了事件处理逻辑
private final EventHandler eventHandler; 
//实际处理事件的线程
private final Thread eventHandlerThread;

public KafkaEventQueue(  
    Time time,  
    LogContext logContext,  
    String threadNamePrefix,  
    Event cleanupEvent  
) {  
    this.time = time;  
    this.cleanupEvent = Objects.requireNonNull(cleanupEvent);  
    this.lock = new ReentrantLock();  
    this.log = logContext.logger(KafkaEventQueue.class);  
    this.eventHandler = new EventHandler();  
    this.eventHandlerThread = new KafkaThread(threadNamePrefix &#43; EVENT_HANDLER_THREAD_SUFFIX,  
        this.eventHandler, false);  
    this.shuttingDown = false;  
    this.interrupted = false;  
    this.eventHandlerThread.start();  
}
```


enqueue操作会将事件存储到一个双向链表，提供EventHandler进行处理。

```java
Exception enqueue(EventContext eventContext,  
                  Function&lt;OptionalLong, OptionalLong&gt; deadlineNsCalculator) {  
    //1.1 上锁，检查是否关闭或中断  
    lock.lock();  
    try {  
        if (shuttingDown) {  
            return new RejectedExecutionException(&#34;The event queue is shutting down&#34;);  
        }  
        if (interrupted) {  
            return new InterruptedException(&#34;The event handler thread is interrupted&#34;);  
        }  
  
        //1.2 检查tag、计算deadline time  
        OptionalLong existingDeadlineNs = OptionalLong.empty();  
        if (eventContext.tag != null) {  
            EventContext toRemove =  
                tagToEventContext.put(eventContext.tag, eventContext);  
            if (toRemove != null) {  
                existingDeadlineNs = toRemove.deadlineNs;  
                remove(toRemove);  
                size--;  
            }  
        }  
        OptionalLong deadlineNs = deadlineNsCalculator.apply(existingDeadlineNs);  
        boolean queueWasEmpty = head.isSingleton();  
        boolean shouldSignal = false;  
        //1.3 根据插入类型，选择插入位置  
        switch (eventContext.insertionType) {  
            case APPEND:  
                head.insertBefore(eventContext);  
                if (queueWasEmpty) {  
                    shouldSignal = true;  
                }  
                break;  
            case PREPEND:  
                head.insertAfter(eventContext);  
                if (queueWasEmpty) {  
                    shouldSignal = true;  
                }  
                break;  
            case DEFERRED:  
                if (!deadlineNs.isPresent()) {  
                    return new RuntimeException(  
                        &#34;You must specify a deadline for deferred events.&#34;);  
                }  
                break;  
        }  
  
        //1.4 如果事件有deadline time，将其插入deadlineMap中，更新size，唤醒等待线程  
        if (deadlineNs.isPresent()) {  
            long insertNs = deadlineNs.getAsLong();  
            long prevStartNs = deadlineMap.isEmpty() ? Long.MAX_VALUE : deadlineMap.firstKey();  
            // If the time in nanoseconds is already taken, take the next one.  
            while (deadlineMap.putIfAbsent(insertNs, eventContext) != null) {  
                insertNs&#43;&#43;;  
            }  
            eventContext.deadlineNs = OptionalLong.of(insertNs);  
            // If the new timeout is before all the existing ones, wake up the  
            // timeout thread.            if (insertNs &lt;= prevStartNs) {  
                shouldSignal = true;  
            }  
        }  
        size&#43;&#43;;  
        if (shouldSignal) {  
            cond.signal();  
        }  
    } finally {  
        lock.unlock();  
    }  
    return null;  
}
```


EventHandler是一个Runnable实现，负责处理进入KafkaEventQueue的各类事件：

```java
@Override  
public void run() {  
    try {  
        handleEvents();  
    } catch (Throwable e) {  
        log.warn(&#34;event handler thread exiting with exception&#34;, e);  
    }  
    try {  
        cleanupEvent.run();  
    } catch (Throwable e) {  
        log.warn(&#34;cleanup event threw exception&#34;, e);  
    }  
}
```


handleEvents方法从双向链表和deadlineMap中不断地取事件处理，事件实际运行的逻辑为：`event.run()`，也就是执行入队列的元素run方法。

```java
private void handleEvents() {  
    Throwable toDeliver = null;  
    EventContext toRun = null;  
    boolean wasInterrupted = false;  
    //1.1 循环处理事件  
    while (true) {  
        //1.2 如果toRun不为空，则运行事件event.run()  
        if (toRun != null) {  
            wasInterrupted = toRun.run(log, toDeliver);  
        }  
  
        //1.3 上锁检查deadlineMap 中是否有超时或延迟事件  
        lock.lock();  
        try {  
            if (toRun != null) {  
                size--;  
                if (wasInterrupted) {  
                    interrupted = wasInterrupted;  
                }  
                toDeliver = null;  
                toRun = null;  
                wasInterrupted = false;  
            }  
            long awaitNs = Long.MAX_VALUE;  
            Map.Entry&lt;Long, EventContext&gt; entry = deadlineMap.firstEntry();  
  
            //1.4 如果deadlineMap不为空，则获取第一个事件  
            if (entry != null) {  
                // Search for timed-out events or deferred events that are ready  
                // to run.                long now = time.nanoseconds();  
                long timeoutNs = entry.getKey();  
                EventContext eventContext = entry.getValue();  
                //1.5 如果第一个事件超时，赋值toRun，在下一轮循环执行  
                if (timeoutNs &lt;= now) {  
                    if (eventContext.insertionType == EventInsertionType.DEFERRED) {  
                        // The deferred event is ready to run.  Prepend it to the  
                        // queue.  (The value for deferred events is a schedule time                        // rather than a timeout.)                        remove(eventContext);  
                        toDeliver = null;  
                        toRun = eventContext;  
                    } else {  
                        // not a deferred event, so it is a deadline, and it is timed out.  
                        remove(eventContext);  
                        toDeliver = new TimeoutException();  
                        toRun = eventContext;  
                    }  
                    continue;  
                } else if (interrupted) {  
                    remove(eventContext);  
                    toDeliver = new InterruptedException(&#34;The event handler thread is interrupted&#34;);  
                    toRun = eventContext;  
                    continue;  
                } else if (shuttingDown) {  
                    remove(eventContext);  
                    toDeliver = new RejectedExecutionException(&#34;The event queue is shutting down&#34;);  
                    toRun = eventContext;  
                    continue;  
                }  
                awaitNs = timeoutNs - now;  
            }  
  
            //2.1 如果队列为空，并且shuttingDown或interrupted，则直接退出循环  
            if (head.next == head) {  
                if (deadlineMap.isEmpty() &amp;&amp; (shuttingDown || interrupted)) {  
                    // If there are no more entries to process, and the queue is  
                    // closing, exit the thread.                    return;  
                }  
            } else {  
                //2.2 否则获取队列的第一个事件，赋值toRun，在下一轮循环执行  
                if (interrupted) {  
                    toDeliver = new InterruptedException(&#34;The event handler thread is interrupted&#34;);  
                } else {  
                    toDeliver = null;  
                }  
                toRun = head.next;  
                remove(toRun);  
                continue;  
            }  
  
            //3.1 wait等待新事件  
            if (awaitNs == Long.MAX_VALUE) {  
                try {  
                    cond.await();  
                } catch (InterruptedException e) {  
                    log.warn(&#34;Interrupted while waiting for a new event. &#34; &#43;  
                        &#34;Shutting down event queue&#34;);  
                    interrupted = true;  
                }  
            } else {  
                try {  
                    cond.awaitNanos(awaitNs);  
                } catch (InterruptedException e) {  
                    log.warn(&#34;Interrupted while waiting for a deferred event. &#34; &#43;  
                        &#34;Shutting down event queue&#34;);  
                    interrupted = true;  
                }  
            }  
        } finally {  
            lock.unlock();  
        }  
    }  
}
```



# 5. Reference

[https://docs.confluent.io/platform/current/kafka-metadata/kraft.html](https://docs.confluent.io/platform/current/kafka-metadata/kraft.html)
[https://cwiki.apache.org/confluence/display/KAFKA/KIP-500](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A&#43;Replace&#43;ZooKeeper&#43;with&#43;a&#43;Self-Managed&#43;Metadata&#43;Quorum)
[https://cwiki.apache.org/confluence/display/KAFKA/KIP-631](https://cwiki.apache.org/confluence/display/KAFKA/KIP-631%3A&#43;The&#43;Quorum-based&#43;Kafka&#43;Controller)
[https://cwiki.apache.org/confluence/display/KAFKA/KIP-595](https://cwiki.apache.org/confluence/display/KAFKA/KIP-595%3A&#43;A&#43;Raft&#43;Protocol&#43;for&#43;the&#43;Metadata&#43;Quorum#KIP595:ARaftProtocolfortheMetadataQuorum-ParentKIP)

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-9-kraft-basic/  

