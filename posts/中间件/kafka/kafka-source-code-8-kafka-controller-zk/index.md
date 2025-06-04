# Kafka源码分析(八)- KafkaController管理(ZK架构)


# 1. Intro

Controller节点作为Kafka集群的管理节点，它负责为所有partition选举leader副本，并且负责存储和管理集群metadata。本文主要对core模块kafka.controller作简要分析。


核心属性如下：
- **controllerContext**：集群metadata对象
- **controllerChannelManager**：Controller端channel管理器，负责向其他Broker发送请求
- **kafkaScheduler**：定时调度器
- **eventManager**：Controller事件管理器，负责管理事件处理线程。
- **replicaStateMachine**：副本状态机，负责副本状态转换。
- **partitionStateMachine**：分区状态机，负责分区状态转换。
- **topicDeletionManager**：主题删除管理器，负责删除主题及日志。

```scala

val controllerContext = new ControllerContext 

var controllerChannelManager = new ControllerChannelManager(  
  () =&gt; controllerContext.epoch,  
  config,  
  time,  
  metrics,  
  stateChangeLogger,  
  threadNamePrefix  
)  
  
// have a separate scheduler for the controller to be able to start and stop independently of the kafka server  
// visible for testing  
private[controller] val kafkaScheduler = new KafkaScheduler(1)  
  
// visible for testing  
private[controller] val eventManager = new ControllerEventManager(config.brokerId, this, time,  
  controllerContext.stats.rateAndTimeMetrics)  
  
private val brokerRequestBatch = new ControllerBrokerRequestBatch(config, controllerChannelManager,  
  eventManager, controllerContext, stateChangeLogger)  
val replicaStateMachine: ReplicaStateMachine = new ZkReplicaStateMachine(config, stateChangeLogger, controllerContext, zkClient,  
  new ControllerBrokerRequestBatch(config, controllerChannelManager, eventManager, controllerContext, stateChangeLogger))  
val partitionStateMachine: PartitionStateMachine = new ZkPartitionStateMachine(config, stateChangeLogger, controllerContext, zkClient,  
  new ControllerBrokerRequestBatch(config, controllerChannelManager, eventManager, controllerContext, stateChangeLogger))  
private val topicDeletionManager = new TopicDeletionManager(config, controllerContext, replicaStateMachine,  
  partitionStateMachine, new ControllerDeletionClient(this, zkClient))
```


# 2. metadata管理


通过kafka.controller.ControllerContext可以看到集群的所有metadata信息，核心字段如下：

- **ControllerStats**：Controller的统计指标
- **offlinePartitionCount**：当前集群中不可用partition数量
- **shuttingDownBrokerIds**：正在关闭的broker id列表
- **liveBrokers**：当前运行中的Broker对象
- **liveBrokerEpochs**：保存运行中Broker的Epoch信息，集群选举时用于判断。
- epoch、epochZkVersion：当前集群controller的epoch值。其中epoch是ZK中`/controller_epoch`节点的值，epochZkVersion是`/controller_epoch`节点的dataVersion值。
- **allTopics**：存储当前集群所有的topic名称
- **partitionAssignments**：保存所有partition副本分配情况，结构为`mutable.Map.empty[String, mutable.Map[Int, ReplicaAssignment]]`，第一层key为topicName，第二层key为partition序号，第二层value为ReplicaAssignment对象



## 2.1. initializeControllerContext

```scala
private def initializeControllerContext(): Unit = {  
  // 1.1 获取所有broker和epoch信息  
  val curBrokerAndEpochs = zkClient.getAllBrokerAndEpochsInCluster  
  val (compatibleBrokerAndEpochs, incompatibleBrokerAndEpochs) = partitionOnFeatureCompatibility(curBrokerAndEpochs)  
  if (incompatibleBrokerAndEpochs.nonEmpty) {  
    warn(&#34;Ignoring registration of new brokers due to incompatibilities with finalized features: &#34; &#43;  
      incompatibleBrokerAndEpochs.map { case (broker, _) =&gt; broker.id }.toSeq.sorted.mkString(&#34;,&#34;))  
  }  
  controllerContext.setLiveBrokers(compatibleBrokerAndEpochs)  
  info(s&#34;Initialized broker epochs cache: ${controllerContext.liveBrokerIdAndEpochs}&#34;)  
  controllerContext.setAllTopics(zkClient.getAllTopicsInCluster(true))  
  registerPartitionModificationsHandlers(controllerContext.allTopics.toSeq)  
  val replicaAssignmentAndTopicIds = zkClient.getReplicaAssignmentAndTopicIdForTopics(controllerContext.allTopics.toSet)  
  processTopicIds(replicaAssignmentAndTopicIds)  
  
  replicaAssignmentAndTopicIds.foreach { case TopicIdReplicaAssignment(_, _, assignments) =&gt;  
    assignments.foreach { case (topicPartition, replicaAssignment) =&gt;  
      controllerContext.updatePartitionFullReplicaAssignment(topicPartition, replicaAssignment)  
      if (replicaAssignment.isBeingReassigned)  
        controllerContext.partitionsBeingReassigned.add(topicPartition)  
    }  
  }  
  controllerContext.clearPartitionLeadershipInfo()  
  controllerContext.shuttingDownBrokerIds.clear()  
  // register broker modifications handlers  
  registerBrokerModificationsHandler(controllerContext.liveOrShuttingDownBrokerIds)  
  // update the leader and isr cache for all existing partitions from Zookeeper  
  updateLeaderAndIsrCache()  
  // start the channel manager  
  controllerChannelManager.startup(controllerContext.liveOrShuttingDownBrokers)  
  info(s&#34;Currently active brokers in the cluster: ${controllerContext.liveBrokerIds}&#34;)  
  info(s&#34;Currently shutting brokers in the cluster: ${controllerContext.shuttingDownBrokerIds}&#34;)  
  info(s&#34;Current list of topics in the cluster: ${controllerContext.allTopics}&#34;)  
}
```


# 3. Controller通信逻辑

ControllerChannelManager用来管理Controller与其他Broker通信channel，与每一个Broker创建通信线程用于消息传递。

其中brokerStateInfo用于存储Broker Id到ControllerBrokerStateInfo结构的映射：
```scala
protected val brokerStateInfo = new mutable.HashMap[Int, ControllerBrokerStateInfo]
```


ControllerBrokerStateInfo结构如下：

```scala
case class ControllerBrokerStateInfo(networkClient: NetworkClient,  
                                     brokerNode: Node,  
                                     messageQueue: BlockingQueue[QueueItem],  
                                     requestSendThread: RequestSendThread,  
                                     queueSizeGauge: Gauge[Int],  
                                     requestRateAndTimeMetrics: Timer,  
                                     reconfigurableChannelBuilder: Option[Reconfigurable])
```

关键字段：
- **brokerNode**：目标Broker节点，封装了Broker的连接信息，比如host、port等信息
- **messageQueue**：请求消息阻塞队列
- **requestSendThread**：Controller使用该线程给目标Broker发送请求

## 3.1. RequestSendThread

RequestSendThread的构造方法如下：

```scala
class RequestSendThread(val controllerId: Int,    //controller的broker id  
                        controllerEpoch: () =&gt; Int,     //当前controller epoch  
                        val queue: BlockingQueue[QueueItem],  //请求阻塞队列  
                        val networkClient: NetworkClient, //执行网络IO的客户端  
                        val brokerNode: Node,     //broker节点  
                        val config: KafkaConfig,    //配置信息  
                        val time: Time,  
                        val requestRateAndQueueTimeMetrics: Timer,  
                        val stateChangeLogger: StateChangeLogger,  
                        name: String)
```


线程处理逻辑如下：

```scala
override def doWork(): Unit = {  
  
  def backoff(): Unit = pause(100, TimeUnit.MILLISECONDS)  
  
  //1.1 从阻塞队列中获取请求  
  val QueueItem(apiKey, requestBuilder, callback, enqueueTimeMs) = queue.take()  
  //1.2 更新监控指标  
  requestRateAndQueueTimeMetrics.update(time.milliseconds() - enqueueTimeMs, TimeUnit.MILLISECONDS)  
  
  var clientResponse: ClientResponse = null  
  try {  
    var isSendSuccessful = false  
    while (isRunning &amp;&amp; !isSendSuccessful) {  
      // if a broker goes down for a long time, then at some point the controller&#39;s zookeeper listener will trigger a  
      // removeBroker which will invoke shutdown() on this thread. At that point, we will stop retrying.      try {  
        //2.1 检查与broker是否已经建立连接  
        if (!brokerReady()) {  
          isSendSuccessful = false  
          backoff()  
        }  
        else {  
          //2.2 发送请求，等待接收response  
          val clientRequest = networkClient.newClientRequest(brokerNode.idString, requestBuilder,  
            time.milliseconds(), true)  
          clientResponse = NetworkClientUtils.sendAndReceive(networkClient, clientRequest, time)  
          isSendSuccessful = true  
        }  
      } catch {  
        case e: Throwable =&gt; // if the send was not successful, reconnect to broker and resend the message  
          warn(s&#34;Controller $controllerId epoch ${controllerEpoch()} fails to send request &#34; &#43;  
            s&#34;$requestBuilder &#34; &#43;  
            s&#34;to broker $brokerNode. Reconnecting to broker.&#34;, e)  
          networkClient.close(brokerNode.idString)  
          isSendSuccessful = false  
          backoff()  
      }  
    }  
    //3.1 如果接收到response  
    if (clientResponse != null) {  
      val requestHeader = clientResponse.requestHeader  
      val api = requestHeader.apiKey  
      //3.2 如果不是leaderAndIsr, stopReplica, updateMetadata请求，则抛出异常  
      if (api != ApiKeys.LEADER_AND_ISR &amp;&amp; api != ApiKeys.STOP_REPLICA &amp;&amp; api != ApiKeys.UPDATE_METADATA)  
        throw new KafkaException(s&#34;Unexpected apiKey received: $apiKey&#34;)  
  
      val response = clientResponse.responseBody  
  
      stateChangeLogger.withControllerEpoch(controllerEpoch()).trace(s&#34;Received response &#34; &#43;  
        s&#34;$response for request $api with correlation id &#34; &#43;  
        s&#34;${requestHeader.correlationId} sent to broker $brokerNode&#34;)  
	  //3.3 调用callback
      if (callback != null) {  
        callback(response)  
      }  
    }  
  } catch {  
    case e: Throwable =&gt;  
      error(s&#34;Controller $controllerId fails to send a request to broker $brokerNode&#34;, e)  
      // If there is any socket error (eg, socket timeout), the connection is no longer usable and needs to be recreated.  
      networkClient.close(brokerNode.idString)  
  }  
}
```


# 4. Controller事件处理逻辑


ControllerEventManager用于处理Controller端事件的管理器，它是事件驱动模式，与前文AsyncKafkaConsumer的思想类似。


## 4.1. ControllerEvent

先来看下ControllerEvent接口，它是对于Controller事件的封装trait：

```scala
sealed trait ControllerEvent {  
  def state: ControllerState  
  // preempt() is not executed by `ControllerEventThread` but by the main thread.  
  def preempt(): Unit  
}
```

```scala
sealed abstract class ControllerState {  
  
  def value: Byte  
  
  def rateAndTimeMetricName: Option[String] =  
    if (hasRateAndTimeMetric) Some(s&#34;${toString}RateAndTimeMs&#34;) else None  
  
  protected def hasRateAndTimeMetric: Boolean = true  
}
```

state为Controller状态，相关事件触发后会更改对应状态。

## 4.2. ControllerEventProcessor

ControllerEventProcessor是用来处理各类Controller事件的处理器，目前实现类只有kafka.controller.KafkaController，它提供了两个基础方法：

```scala
trait ControllerEventProcessor {  
  //接收一个Controller事件，并进行处理
  def process(event: ControllerEvent): Unit  
  //接收Controller事件，并抢占队列，优先处理
  def preempt(event: ControllerEvent): Unit  
}
```


实现类KafkaController中的process方法：

```scala
override def process(event: ControllerEvent): Unit = {  
  try {  
    event match {  
      case event: MockEvent =&gt;  
        // Used only in test cases  
        event.process()  
      case ShutdownEventThread =&gt;  
        error(&#34;Received a ShutdownEventThread event. This type of event is supposed to be handle by ControllerEventThread&#34;)  
      case AutoPreferredReplicaLeaderElection =&gt;  
        processAutoPreferredReplicaLeaderElection()  
      case ReplicaLeaderElection(partitions, electionType, electionTrigger, callback) =&gt;  
        processReplicaLeaderElection(partitions, electionType, electionTrigger, callback)  
      case UncleanLeaderElectionEnable =&gt;  
        processUncleanLeaderElectionEnable()
        }
    }
}
```

## 4.3. ControllerEventManager

ControllerEventManager是处理事件逻辑的管理器，内部存储了事件处理线程、阻塞队列：
```scala
@volatile private var _state: ControllerState = ControllerState.Idle  

private val putLock = new ReentrantLock()  

private val queue = new LinkedBlockingQueue[QueuedEvent]  
// Visible for test  
private[controller] var thread = new ControllerEventThread(ControllerEventThreadName)
```

队列元素QueuedEvent中会存储ControllerEvent，并且定义了process、preempt、awaitProcessing方法，具体是通过调用ControllerEventProcessor对应方法来实现。通过spent变量，标记event是否已经执行。

```scala
class QueuedEvent(val event: ControllerEvent,  
                  val enqueueTimeMs: Long) {  
  private val processingStarted = new CountDownLatch(1)  
  private val spent = new AtomicBoolean(false)  
  
  def process(processor: ControllerEventProcessor): Unit = {  
    if (spent.getAndSet(true))  
      return  
    processingStarted.countDown()  
    processor.process(event)  
  }  
  
  def preempt(processor: ControllerEventProcessor): Unit = {  
    if (spent.getAndSet(true))  
      return  
    processor.preempt(event)  
  }  
  
  def awaitProcessing(): Unit = {  
    processingStarted.await()  
  }  
  
  override def toString: String = {  
    s&#34;QueuedEvent(event=$event, enqueueTimeMs=$enqueueTimeMs)&#34;  
  }  
}
```


put方法用于将ControllerEvent包装后并写入blockingQueue。

```scala
def put(event: ControllerEvent): QueuedEvent = inLock(putLock) {  
  val queuedEvent = new QueuedEvent(event, time.milliseconds())  
  queue.put(queuedEvent)  
  queuedEvent  
}  
  
def clearAndPut(event: ControllerEvent): QueuedEvent = inLock(putLock) {  
  val preemptedEvents = new util.ArrayList[QueuedEvent]()  
  queue.drainTo(preemptedEvents)  
  preemptedEvents.forEach(_.preempt(processor))  
  put(event)  
}
```


## 4.4. ControllerEventThread

ControllerEventThread事件处理线程直接调用QueuedEvent的process方法：

```scala
class ControllerEventThread(name: String)  
  extends ShutdownableThread(  
    name, false, s&#34;[ControllerEventThread controllerId=$controllerId] &#34;)  
    with Logging {  
  
  logIdent = logPrefix  
  
  override def doWork(): Unit = {  
    val dequeued = pollFromEventQueue()  
    dequeued.event match {  
      case ShutdownEventThread =&gt; // The shutting down of the thread has been initiated at this point. Ignore this event.  
      case controllerEvent =&gt;  
        _state = controllerEvent.state  
  
        eventQueueTimeHist.update(time.milliseconds() - dequeued.enqueueTimeMs)  
  
        try {  
          def process(): Unit = dequeued.process(processor)  
  
          rateAndTimeMetrics.get(state) match {  
            case Some(timer) =&gt; timer.time(() =&gt; process())  
            case None =&gt; process()  
          }  
        } catch {  
          case e: Throwable =&gt; error(s&#34;Uncaught error processing event $controllerEvent&#34;, e)  
        }  
  
        _state = ControllerState.Idle  
    }  
  }  
}
```



# 5. 基于Zookeeper选举

Zookeeper作为注册中心时，选举是通过创建`/controller`临时节点实现的，并能通过ZK提供的临时节点watch机制，在controller宕机下线时，触发重新选举。

触发选举的场景有三种：集群刚启动时、`/controller`节点被删除或节点数据变更时。


ControllerChangeHandler继承自ZNodeChangeHandler，这是用于处理Znode变化的callback类：

```scala
class ControllerChangeHandler(eventManager: ControllerEventManager) extends ZNodeChangeHandler {  
  override val path: String = ControllerZNode.path  
  
  override def handleCreation(): Unit = eventManager.put(ControllerChange)  
  override def handleDeletion(): Unit = eventManager.put(Reelect)  
  override def handleDataChange(): Unit = eventManager.put(ControllerChange)  
}
```


elect方法逻辑较为简单，其中调用的onControllerFailover方法会完成context初始化、分区、副本状态机初始化等工作：

```scala
private def elect(): Unit = {  
  activeControllerId = zkClient.getControllerId.getOrElse(-1)  
  /*  
   * We can get here during the initial startup and the handleDeleted ZK callback. Because of the potential race condition,   * it&#39;s possible that the controller has already been elected when we get here. This check will prevent the following   * createEphemeralPath method from getting into an infinite loop if this broker is already the controller.   */  //1.1 检查zk中是否已经有活跃controller  
  if (activeControllerId != -1) {  
    debug(s&#34;Broker $activeControllerId has been elected as the controller, so stopping the election process.&#34;)  
    return  
  }  
  
  try {  
    //1.2 尝试注册controller  
    val (epoch, epochZkVersion) = zkClient.registerControllerAndIncrementControllerEpoch(config.brokerId)  
    controllerContext.epoch = epoch  
    controllerContext.epochZkVersion = epochZkVersion  
    activeControllerId = config.brokerId  
  
    info(s&#34;${config.brokerId} successfully elected as the controller. Epoch incremented to ${controllerContext.epoch} &#34; &#43;  
      s&#34;and epoch zk version is now ${controllerContext.epochZkVersion}&#34;)  
  
    //1.3 调用onControllerFailover()，完成controller初始化动作，如果发生异常则撤销  
    onControllerFailover()  
  } catch {  
    case e: ControllerMovedException =&gt;  
      maybeResign()  
  
      if (activeControllerId != -1)  
        debug(s&#34;Broker $activeControllerId was elected as controller instead of broker ${config.brokerId}&#34;, e)  
      else  
        warn(&#34;A controller has been elected but just resigned, this will result in another round of election&#34;, e)  
    case t: Throwable =&gt;  
      error(s&#34;Error while electing or becoming controller on broker ${config.brokerId}. &#34; &#43;  
        s&#34;Trigger controller movement immediately&#34;, t)  
      triggerControllerMove()  
  }  
}
```

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-8-kafka-controller-zk/  

