# Kafka源码分析(六) - 网络通信模块设计




# Intro

Kafka的网络通信设计如图所示（图片引用自[AutoMQ](https://www.automq.com/blog/understand-kafka-network-communication-and-thread-model)）：

![image.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/20250319145048.png)



其中**SocketServer**负责处理外部连接，并将处理结果封装到Response返回。**KafkaRequestHandlerPool**作为I/O处理线程池，执行请求的具体逻辑。二者之间通过RequestChannel进行交互。


# SocketServer

SocketServer负责处理各个Broker之间的通信channel，采用Reactor处理模型，Acceptor负责从socket接受request，Handler负责处理接收来的request，这也是Kafka中的设计亮点之一。

在实现上，分为data-plane和control-plane，防止数据类请求阻塞控制类请求：

```scala
// data-plane  
private[network] val dataPlaneAcceptors = new ConcurrentHashMap[EndPoint, DataPlaneAcceptor]()  
val dataPlaneRequestChannel = new RequestChannel(maxQueuedRequests, DataPlaneAcceptor.MetricPrefix, time, apiVersionManager.newRequestMetrics)  
// control-plane  
private[network] var controlPlaneAcceptorOpt: Option[ControlPlaneAcceptor] = None  
val controlPlaneRequestChannelOpt: Option[RequestChannel] = config.controlPlaneListenerName.map(_ =&gt;  
  new RequestChannel(20, ControlPlaneAcceptor.MetricPrefix, time, apiVersionManager.newRequestMetrics))

private[network] val processors = new ArrayBuffer[Processor]()
```

从上述定义可以看出control-plane的线程数只有一个，这是因为控制类的请求数量较数据类请求少。


初始化逻辑如下：
```scala
private def createDataPlaneAcceptorAndProcessors(endpoint: EndPoint): Unit = synchronized {  
  if (stopped) {  
    throw new RuntimeException(&#34;Can&#39;t create new data plane acceptor and processors: SocketServer is stopped.&#34;)  
  }  
  val parsedConfigs = config.valuesFromThisConfigWithPrefixOverride(endpoint.listenerName.configPrefix)  
  connectionQuotas.addListener(config, endpoint.listenerName)  
  val isPrivilegedListener = controlPlaneRequestChannelOpt.isEmpty &amp;&amp;  
    config.interBrokerListenerName == endpoint.listenerName  
  val dataPlaneAcceptor = createDataPlaneAcceptor(endpoint, isPrivilegedListener, dataPlaneRequestChannel)  
  config.addReconfigurable(dataPlaneAcceptor)  
  dataPlaneAcceptor.configure(parsedConfigs)  
  dataPlaneAcceptors.put(endpoint, dataPlaneAcceptor)  
  info(s&#34;Created data-plane acceptor and processors for endpoint : ${endpoint.listenerName}&#34;)  
}  
  
private def createControlPlaneAcceptorAndProcessor(endpoint: EndPoint): Unit = synchronized {  
  if (stopped) {  
    throw new RuntimeException(&#34;Can&#39;t create new control plane acceptor and processor: SocketServer is stopped.&#34;)  
  }  
  connectionQuotas.addListener(config, endpoint.listenerName)  
  val controlPlaneAcceptor = createControlPlaneAcceptor(endpoint, controlPlaneRequestChannelOpt.get)  
  controlPlaneAcceptor.addProcessors(1)  
  controlPlaneAcceptorOpt = Some(controlPlaneAcceptor)  
  info(s&#34;Created control-plane acceptor and processor for endpoint : ${endpoint.listenerName}&#34;)  
}
```



## Acceptor

Acceptor线程通过Selector &#43; Channel轮询获取acceptable connection，将接收的连接信息传递给下游Processor处理，作为Runnable任务，每个endpoint指定一个acceptor。


### 初始化


```scala
//缓冲区大小
private val sendBufferSize = config.socketSendBufferBytes  
private val recvBufferSize = config.socketReceiveBufferBytes  
private val listenBacklogSize = config.socketListenBacklogSize  
  
//使用nio selector  
private val nioSelector = NSelector.open()  

private[network] var serverChannel: ServerSocketChannel  = _  
//创建serverChannel  
private[network] val localPort: Int  = if (endPoint.port != 0) {  
  endPoint.port  
} else {  
  serverChannel = openServerSocket(endPoint.host, endPoint.port, listenBacklogSize)  
  val newPort = serverChannel.socket().getLocalPort  
  info(s&#34;Opened wildcard endpoint ${endPoint.host}:$newPort&#34;)  
  newPort  
}  
  
//processor线程池，由Acceptor维护
private[network] val processors = new ArrayBuffer[Processor]()
```


### 创建Socket

```scala
private def openServerSocket(host: String, port: Int, listenBacklogSize: Int): ServerSocketChannel = {  
  //1.1 创建InetSocketAddress  
  val socketAddress =  
    if (Utils.isBlank(host))  
      new InetSocketAddress(port)  
    else  
      new InetSocketAddress(host, port)  
        
  //1.2 开启serverChannel  
  val serverChannel = ServerSocketChannel.open()  
  try {  
    //1.3 设置serverChannel为非阻塞模式  
    serverChannel.configureBlocking(false)  
      
    //1.4 设置serverChannel的接收缓冲区大小  
    if (recvBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)  
      serverChannel.socket().setReceiveBufferSize(recvBufferSize)  
        
    //1.5 绑定serverChannel与InetSocketAddress  
    serverChannel.socket.bind(socketAddress, listenBacklogSize)  
    info(s&#34;Awaiting socket connections on ${socketAddress.getHostString}:${serverChannel.socket.getLocalPort}.&#34;)  
  } catch {  
    case e: SocketException =&gt;  
      Utils.closeQuietly(serverChannel, &#34;server socket&#34;)  
      throw new KafkaException(s&#34;Socket server failed to bind to ${socketAddress.getHostString}:$port: ${e.getMessage}.&#34;, e)  
  }  
  serverChannel  
}
```

start方法逻辑：
```scala
def start(): Unit = synchronized {  
  try {  
    if (!shouldRun.get()) {  
      throw new ClosedChannelException()  
    }  
    //1.1 检查初始化serverChannel  
    if (serverChannel == null) {  
      serverChannel = openServerSocket(endPoint.host, endPoint.port, listenBacklogSize)  
      debug(s&#34;Opened endpoint ${endPoint.host}:${endPoint.port}&#34;)  
    }  
    debug(s&#34;Starting processors for listener ${endPoint.listenerName}&#34;)  
  
    //1.2 启动processors  
    processors.foreach(_.start())  
    debug(s&#34;Starting acceptor thread for listener ${endPoint.listenerName}&#34;)  
      
    //1.3 启动acceptor线程  
    thread.start()  
    startedFuture.complete(null)  
    started.set(true)  
  } catch {  
    case e: ClosedChannelException =&gt;  
      debug(s&#34;Refusing to start acceptor for ${endPoint.listenerName} since the acceptor has already been shut down.&#34;)  
      startedFuture.completeExceptionally(e)  
    case t: Throwable =&gt;  
      error(s&#34;Unable to start acceptor for ${endPoint.listenerName}&#34;, t)  
      startedFuture.completeExceptionally(new RuntimeException(s&#34;Unable to start acceptor for ${endPoint.listenerName}&#34;, t))  
  }  
}
```


### 主体逻辑

作为Runnable继承类，run方法如下：
```scala
override def run(): Unit = {  
  //1.1 将serverChannel 注册到 nioSelector，并设置监听事件为 OP_ACCEPT，用于接收新连接  
  serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)  
  try {  
    //1.2 未调用shutdown时，循环处理connection  
    while (shouldRun.get()) {  
      try {  
        //1.3 接受新连接  
        acceptNewConnections()  
          
        //1.4 检查限流连接  
        closeThrottledConnections()  
      }  
      catch {  
        case e: Throwable =&gt; error(&#34;Error occurred&#34;, e)  
      }  
    }  
  } finally {  
    closeAll()  
  }  
}
```



### 处理连接


Acceptor采用nioSelector处理连接事件：

```scala
private def acceptNewConnections(): Unit = {  
  //1.1 调用select，最多阻塞500ms，等待新的连接事件  
  val ready = nioSelector.select(500)  
  
  //1.2 若存在可处理的事件  
  if (ready &gt; 0) {  
    //1.3 调用selectKeys，获取所有可处理的事件  
    val keys = nioSelector.selectedKeys()  
    val iter = keys.iterator()  
  
    //2.1 迭代处理所有事件  
    while (iter.hasNext &amp;&amp; shouldRun.get()) {  
      try {  
        val key = iter.next  
        iter.remove()  
  
        //2.2 检查是否为可接受连接状态  
        if (key.isAcceptable) {  
          accept(key).foreach { socketChannel =&gt;  
            // Assign the channel to the next processor (using round-robin) to which the  
            // channel can be added without blocking. If newConnections queue is full on            // all processors, block until the last one is able to accept a connection.            var retriesLeft = synchronized(processors.length)  
            var processor: Processor = null  
            do {  
              //2.3 采用轮询方式选取processor，来处理当前connection  
              retriesLeft -= 1  
              processor = synchronized {  
                // adjust the index (if necessary) and retrieve the processor atomically for  
                // correct behaviour in case the number of processors is reduced dynamically                currentProcessorIndex = currentProcessorIndex % processors.length  
                processors(currentProcessorIndex)  
              }  
              currentProcessorIndex &#43;= 1  
            } while (!assignNewConnection(socketChannel, processor, retriesLeft == 0))  
          }  
        } else  
          throw new IllegalStateException(&#34;Unrecognized key state for acceptor thread.&#34;)  
      } catch {  
        case e: Throwable =&gt; error(&#34;Error while accepting connection&#34;, e)  
      }  
    }  
  }  
}
```


将connection转给Processor处理：
```scala
private def assignNewConnection(socketChannel: SocketChannel, processor: Processor, mayBlock: Boolean): Boolean = {  
  if (processor.accept(socketChannel, mayBlock, blockedPercentMeter)) {  
    debug(s&#34;Accepted connection from ${socketChannel.socket.getRemoteSocketAddress} on&#34; &#43;  
      s&#34; ${socketChannel.socket.getLocalSocketAddress} and assigned it to processor ${processor.id},&#34; &#43;  
      s&#34; sendBufferSize [actual|requested]: [${socketChannel.socket.getSendBufferSize}|$sendBufferSize]&#34; &#43;  
      s&#34; recvBufferSize [actual|requested]: [${socketChannel.socket.getReceiveBufferSize}|$recvBufferSize]&#34;)  
    true  
  } else  
    false}
```


### Processor维护

Acceptor中维护了Processor线程池，因此创建、移除Processor逻辑也在其中，创建、移除会同步修改requestChannel中存储的processors：

```scala
  def addProcessors(toCreate: Int): Unit = synchronized {  
    val listenerName = endPoint.listenerName  
    val securityProtocol = endPoint.securityProtocol  
    val listenerProcessors = new ArrayBuffer[Processor]()  
  
    for (_ &lt;- 0 until toCreate) {  
      val processor = newProcessor(socketServer.nextProcessorId(), listenerName, securityProtocol)  
      listenerProcessors &#43;= processor  
      requestChannel.addProcessor(processor)  
  
      if (started.get) {  
        processor.start()  
      }  
    }  
    processors &#43;&#43;= listenerProcessors  
  }  
  
  def newProcessor(id: Int, listenerName: ListenerName, securityProtocol: SecurityProtocol): Processor = {  
    val name = s&#34;${threadPrefix()}-kafka-network-thread-$nodeId-${endPoint.listenerName}-${endPoint.securityProtocol}-$id&#34;  
    new Processor(id,  
                  time,  
                  config.socketRequestMaxBytes,  
                  requestChannel,  
                  connectionQuotas,  
                  config.connectionsMaxIdleMs,  
                  config.failedAuthenticationDelayMs,  
                  listenerName,  
                  securityProtocol,  
                  config,  
                  metrics,  
                  credentialProvider,  
                  memoryPool,  
                  logContext,  
                  Processor.ConnectionQueueSize,  
                  isPrivilegedListener,  
                  apiVersionManager,  
                  name)  
  }  
}

private[network] def removeProcessors(removeCount: Int): Unit = synchronized {  
  // Shutdown `removeCount` processors. Remove them from the processor list first so that no more  
  // connections are assigned. Shutdown the removed processors, closing the selector and its connections.  // The processors are then removed from `requestChannel` and any pending responses to these processors are dropped.  val toRemove = processors.takeRight(removeCount)  
  processors.remove(processors.size - removeCount, removeCount)  
  toRemove.foreach(_.close())  
  toRemove.foreach(processor =&gt; requestChannel.removeProcessor(processor.id))  
}
```


## Processor

Processer负责将连接信息添加到RequestChannel的类中，并负责将Response返回。


Processor中包含三个队列：
- **newConnections**：存储创建的新连接信息SocketChannel，接收到accept方法请求后，会将对象存储到该队列，调用configureNewConnections方法时从该队列中取出SocketChannel。
- **inflightResponses**：将响应的Response信息返回后，存储在该队列中，用于部分Response回调处理。
- **responseQueue**：存储需要进行响应的response对象。

```scala
private val newConnections = new ArrayBlockingQueue[SocketChannel](connectionQueueSize)  
private val inflightResponses = mutable.Map[String, RequestChannel.Response]()  
private val responseQueue = new LinkedBlockingDeque[RequestChannel.Response]()
```


Processor中的selector使用的Kafka自定义Selector：
```scala
protected[network] def createSelector(channelBuilder: ChannelBuilder): KSelector = {  
  channelBuilder match {  
    case reconfigurable: Reconfigurable =&gt; config.addReconfigurable(reconfigurable)  
    case _ =&gt;  
  }  
  new KSelector(  
    maxRequestSize,  
    connectionsMaxIdleMs,  
    failedAuthenticationDelayMs,  
    metrics,  
    time,  
    &#34;socket-server&#34;,  
    metricTags,  
    false,  
    true,  
    channelBuilder,  
    memoryPool,  
    logContext)  
}
```

### accept

首先看下上文提到的accept方法：

```scala
def accept(socketChannel: SocketChannel,  
           mayBlock: Boolean,  
           acceptorIdlePercentMeter: com.yammer.metrics.core.Meter): Boolean = {  
  //1.1 将SocketChannel存储到newConnections队列中  
  val accepted = {  
    if (newConnections.offer(socketChannel)) {  
      true  
      //1.2 如果newConnections队列已满，put进行阻塞等待，并记录acceptorIdlePercentMeter  
    } else if (mayBlock) {  
      val startNs = time.nanoseconds  
      newConnections.put(socketChannel)  
      acceptorIdlePercentMeter.mark(time.nanoseconds() - startNs)  
      true  
    } else  
      false  }  
  //写入成功后唤醒processor线程  
  if (accepted)  
    wakeup()  
  accepted  
}

def wakeup(): Unit = selector.wakeup()
```


### 主体逻辑

```scala
override def run(): Unit = {  
  try {  
  
    //1.1 检查是否shutdown  
    while (shouldRun.get()) {  
      try {  
        // 1.2 创建新连接  
        configureNewConnections()  
        // 1.3 发送Response，并将response放入inflightResponses队列  
        processNewResponses()  
        //1.4 获取SocketChannel中就绪的IO事件  
        poll()  
        //1.5 将Request放入Request队列  
        processCompletedReceives()  
        //1.6 处理Response回调  
        processCompletedSends()  
        //1.7 处理因发送失败而导致的连接断开  
        processDisconnected()  
        //1.8 检查限流连接并关闭  
        closeExcessConnections()  
      } catch {  
        case e: Throwable =&gt; processException(&#34;Processor got uncaught exception.&#34;, e)  
      }  
    }  
  } finally {  
    debug(s&#34;Closing selector - processor $id&#34;)  
    CoreUtils.swallow(closeAll(), this, Level.ERROR)  
  }  
}
```


### configureNewConnections

configureNewConnections

```scala
private def configureNewConnections(): Unit = {  
  var connectionsProcessed = 0  
  //1.1 检查当前是否有未处理的connection并防止超额  
  while (connectionsProcessed &lt; connectionQueueSize &amp;&amp; !newConnections.isEmpty) {  
    val channel = newConnections.poll()  
    try {  
      //1.2 将新的connection注册到selector中，更新计数  
      debug(s&#34;Processor $id listening to new connection from ${channel.socket.getRemoteSocketAddress}&#34;)  
      selector.register(connectionId(channel.socket), channel)  
      connectionsProcessed &#43;= 1  
    } catch {  
      // We explicitly catch all exceptions and close the socket to avoid a socket leak.  
      case e: Throwable =&gt;  
        val remoteAddress = channel.socket.getRemoteSocketAddress  
        // need to close the channel here to avoid a socket leak.  
        connectionQuotas.closeChannel(this, listenerName, channel)  
        processException(s&#34;Processor $id closed connection from $remoteAddress&#34;, e)  
    }  
  }  
}
```



### processNewResponses

该方法将从responseQueue中获取Channel，并将response发送给request方，最后将Response存储到inflightResponses队列供后续使用。


```scala
private def processNewResponses(): Unit = {  
  var currentResponse: RequestChannel.Response = null  
  //1.1 从responseQueue中获取待处理的response  
  while ({currentResponse = dequeueResponse(); currentResponse != null}) {  
    val channelId = currentResponse.request.context.connectionId  
    try {  
      currentResponse match {  
        case response: NoOpResponse =&gt;  
          // There is no response to send to the client, we need to read more pipelined requests  
          // that are sitting in the server&#39;s socket buffer          updateRequestMetrics(response)  
          trace(s&#34;Socket server received empty response to send, registering for read: $response&#34;)  
          tryUnmuteChannel(channelId)  
  
        case response: SendResponse =&gt;  
          sendResponse(response, response.responseSend)  
        case response: CloseConnectionResponse =&gt;  
          updateRequestMetrics(response)  
          trace(&#34;Closing socket connection actively according to the response code.&#34;)  
          close(channelId)  
        case _: StartThrottlingResponse =&gt;  
          handleChannelMuteEvent(channelId, ChannelMuteEvent.THROTTLE_STARTED)  
        case _: EndThrottlingResponse =&gt;  
          handleChannelMuteEvent(channelId, ChannelMuteEvent.THROTTLE_ENDED)  
          tryUnmuteChannel(channelId)  
        case _ =&gt;  
          throw new IllegalArgumentException(s&#34;Unknown response type: ${currentResponse.getClass}&#34;)  
      }  
    } catch {  
      case e: Throwable =&gt;  
        processChannelException(channelId, s&#34;Exception while processing response for $channelId&#34;, e)  
    }  
  }  
}
```


返回Response逻辑：

```scala
protected[network] def sendResponse(response: RequestChannel.Response, responseSend: Send): Unit = {  
  val connectionId = response.request.context.connectionId  
  trace(s&#34;Socket server received response to send to $connectionId, registering for write and sending data: $response&#34;)  

  //1.1 根据id获取channel，更新metric  
  if (channel(connectionId).isEmpty) {  
    warn(s&#34;Attempting to send response via channel for which there is no open connection, connection id $connectionId&#34;)  
    response.request.updateRequestMetrics(0L, response)  
  }  
  //1.2 如果connection可连接，则发送response  
  if (openOrClosingChannel(connectionId).isDefined) {  
    //1.3 发送response  
    selector.send(new NetworkSend(connectionId, responseSend))  
    inflightResponses &#43;= (connectionId -&gt; response)  
  }  
}
```


### poll

调用poll方法检查就绪事件：

```scala
private def poll(): Unit = {  
  val pollTimeout = if (newConnections.isEmpty) 300 else 0  
  try selector.poll(pollTimeout)  
  catch {  
    case e @ (_: IllegalStateException | _: IOException) =&gt;   
  }  
}
```


### processCompletedReceives

processCompletedReceives负责接收待处理消息，并将其转发到RequestChannel。

```scala
private def processCompletedReceives(): Unit = {  
  //1.1 获取已接受的消息，逐一处理  
  selector.completedReceives.forEach { receive =&gt;  
    try {  
      //1.2 获取或关闭channel  
      openOrClosingChannel(receive.source) match {  
        case Some(channel) =&gt;  
          val header = parseRequestHeader(receive.payload)  
          //1.3 检查是否为SASL握手  
          if (header.apiKey == ApiKeys.SASL_HANDSHAKE &amp;&amp; channel.maybeBeginServerReauthentication(receive,  
            () =&gt; time.nanoseconds()))  
            trace(s&#34;Begin re-authentication: $channel&#34;)  
          else {  
            //1.4 检查认证是否过期  
            val nowNanos = time.nanoseconds()  
            if (channel.serverAuthenticationSessionExpired(nowNanos)) {  
              // be sure to decrease connection count and drop any in-flight responses  
              debug(s&#34;Disconnecting expired channel: $channel : $header&#34;)  
              close(channel.id)  
              expiredConnectionsKilledCount.record(null, 1, 0)  
            } else {  
              val connectionId = receive.source  
              val context = new RequestContext(header, connectionId, channel.socketAddress, Optional.of(channel.socketPort()),  
                channel.principal, listenerName, securityProtocol, channel.channelMetadataRegistry.clientInformation,  
                isPrivilegedListener, channel.principalSerde)  
  
              val req = new RequestChannel.Request(processor = id, context = context,  
                startTimeNanos = nowNanos, memoryPool, receive.payload, requestChannel.metrics, None)  
  
              // KIP-511: ApiVersionsRequest is intercepted here to catch the client software name  
              // and version. It is done here to avoid wiring things up to the api layer.              if (header.apiKey == ApiKeys.API_VERSIONS) {  
                val apiVersionsRequest = req.body[ApiVersionsRequest]  
                if (apiVersionsRequest.isValid) {  
                  channel.channelMetadataRegistry.registerClientInformation(new ClientInformation(  
                    apiVersionsRequest.data.clientSoftwareName,  
                    apiVersionsRequest.data.clientSoftwareVersion))  
                }  
              }  
              //1.5 将构造好的Request发送到requestChannel  
              requestChannel.sendRequest(req)  
              selector.mute(connectionId)  
              handleChannelMuteEvent(connectionId, ChannelMuteEvent.REQUEST_RECEIVED)  
            }  
          }  
        case None =&gt;  
          // This should never happen since completed receives are processed immediately after `poll()`  
          throw new IllegalStateException(s&#34;Channel ${receive.source} removed from selector before processing completed receive&#34;)  
      }  
    } catch {  
      // note that even though we got an exception, we can assume that receive.source is valid.  
      // Issues with constructing a valid receive object were handled earlier      case e: Throwable =&gt;  
        processChannelException(receive.source, s&#34;Exception while processing request from ${receive.source}&#34;, e)  
    }  
  }  
  selector.clearCompletedReceives()  
}
```


### processCompletedSends

从inflightResponses取出Response执行对应的回调逻辑：

```scala
private def processCompletedSends(): Unit = {  
  selector.completedSends.forEach { send =&gt;  
    try {  
      val response = inflightResponses.remove(send.destinationId).getOrElse {  
        throw new IllegalStateException(s&#34;Send for ${send.destinationId} completed, but not in `inflightResponses`&#34;)  
      }  
        
      // Invoke send completion callback, and then update request metrics since there might be some  
      // request metrics got updated during callback      response.onComplete.foreach(onComplete =&gt; onComplete(send))  
      updateRequestMetrics(response)  
  
      // Try unmuting the channel. If there was no quota violation and the channel has not been throttled,  
      // it will be unmuted immediately. If the channel has been throttled, it will unmuted only if the throttling      // delay has already passed by now.      handleChannelMuteEvent(send.destinationId, ChannelMuteEvent.RESPONSE_SENT)  
      tryUnmuteChannel(send.destinationId)  
    } catch {  
      case e: Throwable =&gt; processChannelException(send.destinationId,  
        s&#34;Exception while processing completed send to ${send.destinationId}&#34;, e)  
    }  
  }  
  selector.clearCompletedSends()  
}
```



### processDisconnected

processDisconnected用于处理已断开连接：

```scala
private def processDisconnected(): Unit = {  
  selector.disconnected.keySet.forEach { connectionId =&gt;  
    try {  
      val remoteHost = ConnectionId.fromString(connectionId).getOrElse {  
        throw new IllegalStateException(s&#34;connectionId has unexpected format: $connectionId&#34;)  
      }.remoteHost  
      inflightResponses.remove(connectionId).foreach(updateRequestMetrics)  
      // the channel has been closed by the selector but the quotas still need to be updated  
      connectionQuotas.dec(listenerName, InetAddress.getByName(remoteHost))  
    } catch {  
      case e: Throwable =&gt; processException(s&#34;Exception while processing disconnection of $connectionId&#34;, e)  
    }  
  }  
}
```


# RequestChannel

**RequestChannel**负责接收Processor传递过来的Request请求，并传递给KafkaRequestHandler进行处理，核心属性如下：

```scala
//请求队列  
private val requestQueue = new ArrayBlockingQueue[BaseRequest](queueSize)  
//SocketServer中的processor  
private val processors = new ConcurrentHashMap[Int, Processor]()  
  
private val requestQueueSizeMetricName = metricNamePrefix.concat(RequestQueueSizeMetric)  
private val responseQueueSizeMetricName = metricNamePrefix.concat(ResponseQueueSizeMetric)  
private val callbackQueue = new ArrayBlockingQueue[BaseRequest](queueSize)
```


```scala
def sendRequest(request: RequestChannel.Request): Unit = {  
  requestQueue.put(request)  
}

def receiveRequest(timeout: Long): RequestChannel.BaseRequest = {  
  val callbackRequest = callbackQueue.poll()  
  if (callbackRequest != null)  
    callbackRequest  
  else {  
    val request = requestQueue.poll(timeout, TimeUnit.MILLISECONDS)  
    request match {  
      case WakeupRequest =&gt; callbackQueue.poll()  
      case _ =&gt; request  
    }  
  }  
}
```


# KafkaRequestHandlerPool


 **KafkaRequestHandlerPool**是请求I/O处理线程池，负责创建、维护、销毁KafkaRequestHandler，**KafkaRequestHandler**作为请求I/O处理线程类，负责从SocketServer的RequestChannel的请求队列中获取请求对象，并处理。


```scala
class KafkaRequestHandlerPool(  
  val brokerId: Int,  
  val requestChannel: RequestChannel,  
  val apis: ApiRequestHandler,  
  time: Time,  
  numThreads: Int,  
  requestHandlerAvgIdleMetricName: String,  
  logAndThreadNamePrefix : String,  
  nodeName: String = &#34;broker&#34;  
) extends Logging {  
  private val metricsGroup = new KafkaMetricsGroup(this.getClass)  
  
  val threadPoolSize: AtomicInteger = new AtomicInteger(numThreads)  
  /* a meter to track the average free capacity of the request handlers */  
  private val aggregateIdleMeter = metricsGroup.newMeter(requestHandlerAvgIdleMetricName, &#34;percent&#34;, TimeUnit.NANOSECONDS)  
  
  this.logIdent = &#34;[&#34; &#43; logAndThreadNamePrefix &#43; &#34; Kafka Request Handler on Broker &#34; &#43; brokerId &#43; &#34;], &#34;  
  val runnables = new mutable.ArrayBuffer[KafkaRequestHandler](numThreads)  
  for (i &lt;- 0 until numThreads) {  
    createHandler(i)  
  }
}

```

初始化步骤位于：kafka.server.BrokerServer#startup / kafka.server.KafkaServer#startup

```scala
dataPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.nodeId,  
  socketServer.dataPlaneRequestChannel, dataPlaneRequestProcessor, time,  
  config.numIoThreads, s&#34;${DataPlaneAcceptor.MetricPrefix}RequestHandlerAvgIdlePercent&#34;,  
  DataPlaneAcceptor.ThreadPrefix)
```

可以看出KafkaRequestHandlerPool中的I/O处理线程数量，是由配置项`num.io.threads`确定。


## 线程池管理


创建线程：

```scala
def createHandler(id: Int): Unit = synchronized {  
  runnables &#43;= new KafkaRequestHandler(id, brokerId, aggregateIdleMeter, threadPoolSize, requestChannel, apis, time, nodeName)  
  KafkaThread.daemon(logAndThreadNamePrefix &#43; &#34;-kafka-request-handler-&#34; &#43; id, runnables(id)).start()  
}
```


线程池扩缩容：

```scala
def resizeThreadPool(newSize: Int): Unit = synchronized {  
  val currentSize = threadPoolSize.get  
  info(s&#34;Resizing request handler thread pool size from $currentSize to $newSize&#34;)  
  if (newSize &gt; currentSize) {  
    for (i &lt;- currentSize until newSize) {  
      createHandler(i)  
    }  
  } else if (newSize &lt; currentSize) {  
    for (i &lt;- 1 to (currentSize - newSize)) {  
      runnables.remove(currentSize - i).stop()  
    }  
  }  
  threadPoolSize.set(newSize)  
}
```


shutdown：

```scala
def shutdown(): Unit = synchronized {  
  info(&#34;shutting down&#34;)  
  for (handler &lt;- runnables)  
    handler.initiateShutdown()  
  for (handler &lt;- runnables)  
    handler.awaitShutdown()  
  info(&#34;shut down completely&#34;)  
}
```



## KafkaRequestHandler

KafkaRequestHandler作为Runnable对象，内部属性如下：

- threadRequestChannel：存储当前绑定的RequestChannel
- threadCurrentRequest：存储当前循环处理的Request信息

```scala
class KafkaRequestHandler(  
  id: Int,  
  brokerId: Int,  
  val aggregateIdleMeter: Meter,  
  val totalHandlerThreads: AtomicInteger,  
  val requestChannel: RequestChannel,  
  apis: ApiRequestHandler,  
  time: Time,  
  nodeName: String = &#34;broker&#34;  
) extends Runnable with Logging {  
  this.logIdent = s&#34;[Kafka Request Handler $id on ${nodeName.capitalize} $brokerId], &#34;  
  private val shutdownComplete = new CountDownLatch(1)  
  private val requestLocal = RequestLocal.withThreadConfinedCaching  
  @volatile private var stopped = false

}

object KafkaRequestHandler {  
  // Support for scheduling callbacks on a request thread.  
  private val threadRequestChannel = new ThreadLocal[RequestChannel]  
  private val threadCurrentRequest = new ThreadLocal[RequestChannel.Request]
}
```


### 主体逻辑

从run方法中可以看出，处理的请求分为三类：shutdown请求、callback请求、普通请求。普通请求通过KafkaApis.handle方法处理。

```scala
def run(): Unit = {  
  //1.1 设置当前线程的RequestChannel  
  threadRequestChannel.set(requestChannel)  
  
  //1.2 线程未关闭的情况，循环处理请求  
  while (!stopped) {  

    //1.3 从requestChannel中获取请求  
    val req = requestChannel.receiveRequest(300)  
    val endTime = time.nanoseconds  
    val idleTime = endTime - startSelectTime  
    aggregateIdleMeter.mark(idleTime / totalHandlerThreads.get)  
  
    req match {  
  
      //2.1 处理shutdown请求  
      case RequestChannel.ShutdownRequest =&gt;  
        debug(s&#34;Kafka request handler $id on broker $brokerId received shut down command&#34;)  
        completeShutdown()  
        return  
  
      //2.2 处理回调请求  
      case callback: RequestChannel.CallbackRequest =&gt;  
        val originalRequest = callback.originalRequest  
        try {  
  
          if (originalRequest.callbackRequestDequeueTimeNanos.isDefined) {  
            val prevCallbacksTimeNanos = originalRequest.callbackRequestCompleteTimeNanos.getOrElse(0L) - originalRequest.callbackRequestDequeueTimeNanos.getOrElse(0L)  
            originalRequest.callbackRequestCompleteTimeNanos = None  
            originalRequest.callbackRequestDequeueTimeNanos = Some(time.nanoseconds() - prevCallbacksTimeNanos)  
          } else {  
            originalRequest.callbackRequestDequeueTimeNanos = Some(time.nanoseconds())  
          }  
            
          threadCurrentRequest.set(originalRequest)  
          callback.fun(requestLocal)  
        } catch {  
          case e: FatalExitError =&gt;  
            completeShutdown()  
            Exit.exit(e.statusCode)  
          case e: Throwable =&gt; error(&#34;Exception when handling request&#34;, e)  
        } finally {  
          // When handling requests, we try to complete actions after, so we should try to do so here as well.  
          apis.tryCompleteActions()  
          if (originalRequest.callbackRequestCompleteTimeNanos.isEmpty)  
            originalRequest.callbackRequestCompleteTimeNanos = Some(time.nanoseconds())  
          threadCurrentRequest.remove()  
        }  
  
        //2.3 处理普通请求  
      case request: RequestChannel.Request =&gt;  
        try {  
          //2.4 由apis处理请求  
          request.requestDequeueTimeNanos = endTime  
          trace(s&#34;Kafka request handler $id on broker $brokerId handling request $request&#34;)  
          threadCurrentRequest.set(request)  
          apis.handle(request, requestLocal)  
        } catch {  
          case e: FatalExitError =&gt;  
            completeShutdown()  
            Exit.exit(e.statusCode)  
          case e: Throwable =&gt; error(&#34;Exception when handling request&#34;, e)  
        } finally {  
          // 移除thread local信息  
          threadCurrentRequest.remove()  
          request.releaseBuffer()  
        }  
  
      case RequestChannel.WakeupRequest =&gt;   
        // We should handle this in receiveRequest by polling callbackQueue.  
        warn(&#34;Received a wakeup request outside of typical usage.&#34;)  
  
      case null =&gt; // continue  
    }  
  }
```

# Reference

[https://www.automq.com/blog/understand-kafka-network-communication-and-thread-model](https://www.automq.com/blog/understand-kafka-network-communication-and-thread-model)

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-6-network-model/  

