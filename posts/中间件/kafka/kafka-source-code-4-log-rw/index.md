# Kafka源码分析(四) - 日志读写流程



# Intro

上文介绍了Kafka日志加载的基本流程，本文主要分析Kafka日志的读写流程。

日志写入的场景主要有以下几种：
1. 生产者向Leader replica写入消息（包含事务消息）
2. Follow replica拉取消息并写入
3. 消费者组写入组信息


# 日志写入


从KafkaApis入口类中可以看到produce的处理动作：
```scala
case ApiKeys.PRODUCE =&gt; handleProduceRequest(request, requestLocal)
```

这里重点关注日志的写入逻辑，handleProduceRequest()中会调用replicaManager执行append：

```scala
// call the replica manager to append messages to the replicas  
replicaManager.handleProduceAppend(  
  timeout = produceRequest.timeout.toLong,  
  requiredAcks = produceRequest.acks,  
  internalTopicsAllowed = internalTopicsAllowed,  
  transactionalId = produceRequest.transactionalId,  
  entriesPerPartition = authorizedRequestInfo,  
  responseCallback = sendResponseCallback,  
  recordValidationStatsCallback = processingStatsCallback,  
  requestLocal = requestLocal,  
  transactionSupportedOperation = transactionSupportedOperation)
```


## appendRecords

在ReplicaManager中会调用appendRecords()将其追加到消息对应partition的leader replica中，并且等待leader将其同步到其他replica中。


```scala
/**  
 * Handles the produce request by starting any transactional verification before appending. * * @param timeout                       maximum time we will wait to append before returning  
 * @param requiredAcks                  number of replicas who must acknowledge the append before sending the response  
 * @param internalTopicsAllowed         boolean indicating whether internal topics can be appended to  
 * @param transactionalId               the transactional ID for the produce request or null if there is none.  
 * @param entriesPerPartition           the records per partition to be appended  
 * @param responseCallback              callback for sending the response  
 * @param recordValidationStatsCallback callback for updating stats on record conversions  
 * @param requestLocal                  container for the stateful instances scoped to this request -- this must correspond to the  
 *                                      thread calling this method * @param actionQueue                   the action queue to use. ReplicaManager#defaultActionQueue is used by default.  
 * @param transactionSupportedOperation determines the supported Operation based on the client&#39;s Request api version  
 * * The responseCallback is wrapped so that it is scheduled on a request handler thread. There, it should be called with * that request handler thread&#39;s thread local and not the one supplied to this method. */
 def handleProduceAppend(timeout: Long,  
                        requiredAcks: Short,  
                        internalTopicsAllowed: Boolean,  
                        transactionalId: String,  
                        entriesPerPartition: Map[TopicPartition, MemoryRecords],  
                        responseCallback: Map[TopicPartition, PartitionResponse] =&gt; Unit,  
                        recordValidationStatsCallback: Map[TopicPartition, RecordValidationStats] =&gt; Unit = _ =&gt; (),  
                        requestLocal: RequestLocal = RequestLocal.NoCaching,  
                        actionQueue: ActionQueue = this.defaultActionQueue,  
                        transactionSupportedOperation: TransactionSupportedOperation): Unit = {
	
	appendRecords( 
	  timeout = timeout,  
	  requiredAcks = requiredAcks,  
	  internalTopicsAllowed = internalTopicsAllowed,  
	  origin = AppendOrigin.CLIENT,  
	  entriesPerPartition = entriesWithoutErrorsPerPartition,  
	  responseCallback = newResponseCallback,  
	  recordValidationStatsCallback = recordValidationStatsCallback,  
	  requestLocal = newRequestLocal,  
	  actionQueue = actionQueue,  
	  verificationGuards = verificationGuards  
		)
}
```

传入的参数较多，重要参数的含义如下：
- timeout：producer端的`request.timeout.ms`参数
- requiredAcks：producer端的`acks`参数
- internalTopicsAllowed：是否允许向内部主题写入消息
- origin：写入方来源，共有四类：REPLICATION（来自leader replica同步的数据）、COORDINATOR（coordinator写入的数据）、CLIENT（客户端写入的数据）、RAFT_LEADER（来自raft leader写入的数据）
- entriesPerPartition：需要写入的消息分组，结构为：`Map[TopicPartition, MemoryRecords]`
- responseCallback：写入成功后的callback


在appendRecords()中会调用appendToLocalLog()将消息写入local log。

```scala
/**  
 * Append messages to leader replicas of the partition, and wait for them to be replicated to other replicas; * the callback function will be triggered either when timeout or the required acks are satisfied; * if the callback function itself is already synchronized on some object then pass this object to avoid deadlock. * * Noted that all pending delayed check operations are stored in a queue. All callers to ReplicaManager.appendRecords() * are expected to call ActionQueue.tryCompleteActions for all affected partitions, without holding any conflicting * locks. * * @param timeout                       maximum time we will wait to append before returning  
 * @param requiredAcks                  number of replicas who must acknowledge the append before sending the response  
 * @param internalTopicsAllowed         boolean indicating whether internal topics can be appended to  
 * @param origin                        source of the append request (ie, client, replication, coordinator)  
 * @param entriesPerPartition           the records per partition to be appended  
 * @param responseCallback              callback for sending the response  
 * @param delayedProduceLock            lock for the delayed actions  
 * @param recordValidationStatsCallback callback for updating stats on record conversions  
 * @param requestLocal                  container for the stateful instances scoped to this request -- this must correspond to the  
 *                                      thread calling this method * @param actionQueue                   the action queue to use. ReplicaManager#defaultActionQueue is used by default.  
 * @param verificationGuards            the mapping from topic partition to verification guards if transaction verification is used  
 */
 def appendRecords(timeout: Long,  
                  requiredAcks: Short,  
                  internalTopicsAllowed: Boolean,  
                  origin: AppendOrigin,  
                  entriesPerPartition: Map[TopicPartition, MemoryRecords],  
                  responseCallback: Map[TopicPartition, PartitionResponse] =&gt; Unit,  
                  delayedProduceLock: Option[Lock] = None,  
                  recordValidationStatsCallback: Map[TopicPartition, RecordValidationStats] =&gt; Unit = _ =&gt; (),  
                  requestLocal: RequestLocal = RequestLocal.NoCaching,  
                  actionQueue: ActionQueue = this.defaultActionQueue,  
                  verificationGuards: Map[TopicPartition, VerificationGuard] = Map.empty): Unit = {  
  //1.1 检查ack参数是否合法  
  if (!isValidRequiredAcks(requiredAcks)) {  
    sendInvalidRequiredAcksResponse(entriesPerPartition, responseCallback)  
    return  
  }  
  
  //1.2 调用appendToLocalLog写入local log  
  val sTime = time.milliseconds  
  val localProduceResults = appendToLocalLog(internalTopicsAllowed = internalTopicsAllowed,  
    origin, entriesPerPartition, requiredAcks, requestLocal, verificationGuards.toMap)  
  debug(&#34;Produce to local log in %d ms&#34;.format(time.milliseconds - sTime))  
  
  //1.3 构建ProducePartitionStatus  
  val produceStatus = buildProducePartitionStatus(localProduceResults)  
  
  //1.4 将写入完成的响应结果localProduceResults，加到actionQueue，如果leader high watermark发生变更，会触发一些延迟操作  
  addCompletePurgatoryAction(actionQueue, localProduceResults)  
  recordValidationStatsCallback(localProduceResults.map { case (k, v) =&gt;  
    k -&gt; v.info.recordValidationStats  
  })  
  
  //1.5 检查是否需要执行延迟操作  
  maybeAddDelayedProduce(  
    requiredAcks,  
    delayedProduceLock,  
    timeout,  
    entriesPerPartition,  
    localProduceResults,  
    produceStatus,  
    responseCallback  
  )  
}
```



## appendToLocalLog

该方法中会获取到消息对应的Partition对象，并调用appendRecordsToLeader()完成写入。

```scala
private def appendToLocalLog(internalTopicsAllowed: Boolean,  
                             origin: AppendOrigin,  
                             entriesPerPartition: Map[TopicPartition, MemoryRecords],  
                             requiredAcks: Short,  
                             requestLocal: RequestLocal,  
                             verificationGuards: Map[TopicPartition, VerificationGuard]): Map[TopicPartition, LogAppendResult] = {  
  
  entriesPerPartition.map { case (topicPartition, records) =&gt;  
    brokerTopicStats.topicStats(topicPartition.topic).totalProduceRequestRate.mark()  
    brokerTopicStats.allTopicsStats.totalProduceRequestRate.mark()  
  
    // reject appending to internal topics if it is not allowed  
    if (Topic.isInternal(topicPartition.topic) &amp;&amp; !internalTopicsAllowed) {  
      (topicPartition, LogAppendResult(  
        LogAppendInfo.UNKNOWN_LOG_APPEND_INFO,  
        Some(new InvalidTopicException(s&#34;Cannot append to internal topic ${topicPartition.topic}&#34;)),  
        hasCustomErrorMessage = false))  
    } else {  
      //1.1 获取partition对象  
      try {  
        val partition = getPartitionOrException(topicPartition)  
        //1.2 向该分区对象写入消息集合  
        val info = partition.appendRecordsToLeader(records, origin, requiredAcks, requestLocal,  
          verificationGuards.getOrElse(topicPartition, VerificationGuard.SENTINEL))  
        val numAppendedMessages = info.numMessages  
  
  
        //1.3 返回写入结果  
        // update stats for successfully appended bytes and messages as bytesInRate and messageInRate  
        brokerTopicStats.topicStats(topicPartition.topic).bytesInRate.mark(records.sizeInBytes)  
        brokerTopicStats.allTopicsStats.bytesInRate.mark(records.sizeInBytes)  
        brokerTopicStats.topicStats(topicPartition.topic).messagesInRate.mark(numAppendedMessages)  
        brokerTopicStats.allTopicsStats.messagesInRate.mark(numAppendedMessages)
    }  
  }  
}
```


```scala
def appendRecordsToLeader(records: MemoryRecords, origin: AppendOrigin, requiredAcks: Int,  
                          requestLocal: RequestLocal, verificationGuard: VerificationGuard = VerificationGuard.SENTINEL): LogAppendInfo = {  
  //1.1 加读锁  
  val (info, leaderHWIncremented) = inReadLock(leaderIsrUpdateLock) {  
    // 1.2 判断是否为leader副本  
    leaderLogIfLocal match {  
      case Some(leaderLog) =&gt;  
        //1.3 获取最小ISR  
        val minIsr = effectiveMinIsr(leaderLog)  
        val inSyncSize = partitionState.isr.size  
  
        // Avoid writing to leader if there are not enough insync replicas to make it safe  
        if (inSyncSize &lt; minIsr &amp;&amp; requiredAcks == -1) {  
          throw new NotEnoughReplicasException(s&#34;The size of the current ISR ${partitionState.isr} &#34; &#43;  
            s&#34;is insufficient to satisfy the min.isr requirement of $minIsr for partition $topicPartition&#34;)  
        }  
  
        // 1.4 调用UnifiedLog的appendAsLeader方法进行写入  
        val info = leaderLog.appendAsLeader(records, leaderEpoch = this.leaderEpoch, origin,  
          interBrokerProtocolVersion, requestLocal, verificationGuard)  
  
        // we may need to increment high watermark since ISR could be down to 1  
        (info, maybeIncrementLeaderHW(leaderLog))  
  
      case None =&gt;  
        throw new NotLeaderOrFollowerException(&#34;Leader not local for partition %s on broker %d&#34;  
          .format(topicPartition, localBrokerId))  
    }  
  }  
  
  info.copy(if (leaderHWIncremented) LeaderHwChange.INCREASED else LeaderHwChange.SAME)  
}
```

```scala
def appendAsLeader(records: MemoryRecords,  
                   leaderEpoch: Int,  
                   origin: AppendOrigin = AppendOrigin.CLIENT,  
                   interBrokerProtocolVersion: MetadataVersion = MetadataVersion.latestProduction,  
                   requestLocal: RequestLocal = RequestLocal.NoCaching,  
                   verificationGuard: VerificationGuard = VerificationGuard.SENTINEL): LogAppendInfo = {  
  val validateAndAssignOffsets = origin != AppendOrigin.RAFT_LEADER  
  append(records, origin, interBrokerProtocolVersion, validateAndAssignOffsets, leaderEpoch, Some(requestLocal), verificationGuard, ignoreRecordSize = false)  
}
```


先看下UnifiedLog的append方法参数：
- records：待插入的消息记录
- origin：上文提到的消息插入来源
- interBrokerProtocolVersion：broker消息版本
- validateAndAssignOffsets：是否需要校验并给消息分配offset
- leaderEpoch：leader选举版本
- requstLocal：用于存储请求信息的有状态本地容器
- ignoreRecordSize：是否忽略校验大小

方法内部首先做数据校验，分配offset，再通过LocalLog进行写入。

```scala
private def append(records: MemoryRecords,  
                   origin: AppendOrigin,  
                   interBrokerProtocolVersion: MetadataVersion,  
                   validateAndAssignOffsets: Boolean,  
                   leaderEpoch: Int,  
                   requestLocal: Option[RequestLocal],  
                   verificationGuard: VerificationGuard,  
                   ignoreRecordSize: Boolean): LogAppendInfo = {  
  // We want to ensure the partition metadata file is written to the log dir before any log data is written to disk.  
  // This will ensure that any log data can be recovered with the correct topic ID in the case of failure.  //1.1 将metadata文件刷入磁盘  
  maybeFlushMetadataFile()  
  
  //1.2 校验消息  
  val appendInfo = analyzeAndValidateRecords(records, origin, ignoreRecordSize, !validateAndAssignOffsets, leaderEpoch)  
  
  // return if we have no valid messages or if this is a duplicate of the last appended entry  
  if (appendInfo.validBytes &lt;= 0) appendInfo  
  else {  
  
    //1.3 清除消息中的不合法字节  
    // trim any invalid bytes or partial messages before appending it to the on-disk log  
    var validRecords = trimInvalidBytes(records, appendInfo)  
  
  
    //1.4 设置同步，执行写入  
    // they are valid, insert them in the log  
    lock synchronized {  
      maybeHandleIOException(s&#34;Error while appending records to $topicPartition in dir ${dir.getParent}&#34;) {  
        localLog.checkIfMemoryMappedBufferClosed()  
  
        //1.5 检查是否需要手动分配offset  
        if (validateAndAssignOffsets) {  
          // assign offsets to the message set  
          val offset = PrimitiveRef.ofLong(localLog.logEndOffset)  
          appendInfo.setFirstOffset(offset.value)  
          val validateAndOffsetAssignResult = try {  
            val targetCompression = BrokerCompressionType.targetCompression(config.compression, appendInfo.sourceCompression())  
            val validator = new LogValidator(validRecords,  
              topicPartition,  
              time,  
              appendInfo.sourceCompression,  
              targetCompression,  
              config.compact,  
              config.recordVersion.value,  
              config.messageTimestampType,  
              config.messageTimestampBeforeMaxMs,  
              config.messageTimestampAfterMaxMs,  
              leaderEpoch,  
              origin,  
              interBrokerProtocolVersion  
            )  
            validator.validateMessagesAndAssignOffsets(offset,  
              validatorMetricsRecorder,  
              requestLocal.getOrElse(throw new IllegalArgumentException(  
                &#34;requestLocal should be defined if assignOffsets is true&#34;)  
              ).bufferSupplier  
            )  
          } catch {  
            case e: IOException =&gt;  
              throw new KafkaException(s&#34;Error validating messages while appending to log $name&#34;, e)  
          }  
  
          validRecords = validateAndOffsetAssignResult.validatedRecords  
          appendInfo.setMaxTimestamp(validateAndOffsetAssignResult.maxTimestampMs)  
          appendInfo.setShallowOffsetOfMaxTimestamp(validateAndOffsetAssignResult.shallowOffsetOfMaxTimestamp)  
          appendInfo.setLastOffset(offset.value - 1)  
          appendInfo.setRecordValidationStats(validateAndOffsetAssignResult.recordValidationStats)  
          if (config.messageTimestampType == TimestampType.LOG_APPEND_TIME)  
            appendInfo.setLogAppendTime(validateAndOffsetAssignResult.logAppendTimeMs)  
  
          // re-validate message sizes if there&#39;s a possibility that they have changed (due to re-compression or message  
          // format conversion)          if (!ignoreRecordSize &amp;&amp; validateAndOffsetAssignResult.messageSizeMaybeChanged) {  
            validRecords.batches.forEach { batch =&gt;  
              if (batch.sizeInBytes &gt; config.maxMessageSize) {  
                // we record the original message set size instead of the trimmed size  
                // to be consistent with pre-compression bytesRejectedRate recording                brokerTopicStats.topicStats(topicPartition.topic).bytesRejectedRate.mark(records.sizeInBytes)  
                brokerTopicStats.allTopicsStats.bytesRejectedRate.mark(records.sizeInBytes)  
                throw new RecordTooLargeException(s&#34;Message batch size is ${batch.sizeInBytes} bytes in append to&#34; &#43;  
                  s&#34;partition $topicPartition which exceeds the maximum configured size of ${config.maxMessageSize}.&#34;)  
              }  
            }  
          }  
        } else {  
          // 从appendInfo获取offset  
          // we are taking the offsets we are given          if (appendInfo.firstOrLastOffsetOfFirstBatch &lt; localLog.logEndOffset) {  
            // we may still be able to recover if the log is empty  
            // one example: fetching from log start offset on the leader which is not batch aligned,            // which may happen as a result of AdminClient#deleteRecords()            val hasFirstOffset = appendInfo.firstOffset != UnifiedLog.UnknownOffset  
            val firstOffset = if (hasFirstOffset) appendInfo.firstOffset else records.batches.iterator().next().baseOffset()  
  
            val firstOrLast = if (hasFirstOffset) &#34;First offset&#34; else &#34;Last offset of the first batch&#34;  
            throw new UnexpectedAppendOffsetException(  
              s&#34;Unexpected offset in append to $topicPartition. $firstOrLast &#34; &#43;  
                s&#34;${appendInfo.firstOrLastOffsetOfFirstBatch} is less than the next offset ${localLog.logEndOffset}. &#34; &#43;  
                s&#34;First 10 offsets in append: ${records.records.asScala.take(10).map(_.offset)}, last offset in&#34; &#43;  
                s&#34; append: ${appendInfo.lastOffset}. Log start offset = $logStartOffset&#34;,  
              firstOffset, appendInfo.lastOffset)  
          }  
        }  
  
        // update the epoch cache with the epoch stamped onto the message by the leader  
        validRecords.batches.forEach { batch =&gt;  
          if (batch.magic &gt;= RecordBatch.MAGIC_VALUE_V2) {  
            maybeAssignEpochStartOffset(batch.partitionLeaderEpoch, batch.baseOffset)  
          } else {  
            // In partial upgrade scenarios, we may get a temporary regression to the message format. In  
            // order to ensure the safety of leader election, we clear the epoch cache so that we revert            // to truncation by high watermark after the next leader election.            leaderEpochCache.filter(_.nonEmpty).foreach { cache =&gt;  
              warn(s&#34;Clearing leader epoch cache after unexpected append with message format v${batch.magic}&#34;)  
              cache.clearAndFlush()  
            }  
          }  
        }  
  
        // check messages set size may be exceed config.segmentSize  
        if (validRecords.sizeInBytes &gt; config.segmentSize) {  
          throw new RecordBatchTooLargeException(s&#34;Message batch size is ${validRecords.sizeInBytes} bytes in append &#34; &#43;  
            s&#34;to partition $topicPartition, which exceeds the maximum configured segment size of ${config.segmentSize}.&#34;)  
        }  
  
        // 2.1 检查LogSegment是否已满，并返回对应的LogSegment  
        // maybe roll the log if this segment is full        val segment = maybeRoll(validRecords.sizeInBytes, appendInfo)  
  
        val logOffsetMetadata = new LogOffsetMetadata(  
          appendInfo.firstOrLastOffsetOfFirstBatch,  
          segment.baseOffset,  
          segment.size)  
  
        //2.2 检查事务状态  
        // now that we have valid records, offsets assigned, and timestamps updated, we need to  
        // validate the idempotent/transactional state of the producers and collect some metadata        val (updatedProducers, completedTxns, maybeDuplicate) = analyzeAndValidateProducerState(  
          logOffsetMetadata, validRecords, origin, verificationGuard)  
  
        maybeDuplicate match {  
          case Some(duplicate) =&gt;  
            appendInfo.setFirstOffset(duplicate.firstOffset)  
            appendInfo.setLastOffset(duplicate.lastOffset)  
            appendInfo.setLogAppendTime(duplicate.timestamp)  
            appendInfo.setLogStartOffset(logStartOffset)  
          case None =&gt;  
            // Append the records, and increment the local log end offset immediately after the append because a  
            // write to the transaction index below may fail and we want to ensure that the offsets            // of future appends still grow monotonically. The resulting transaction index inconsistency            // will be cleaned up after the log directory is recovered. Note that the end offset of the            // ProducerStateManager will not be updated and the last stable offset will not advance            // if the append to the transaction index fails.  
            //2.3 调用append()方法执行写入，并更新localLog.logEndOffset和highWatermark  
            localLog.append(appendInfo.lastOffset, appendInfo.maxTimestamp, appendInfo.shallowOffsetOfMaxTimestamp, validRecords)  
            updateHighWatermarkWithLogEndOffset()  
  
            // update the producer state  
            updatedProducers.values.foreach(producerAppendInfo =&gt; producerStateManager.update(producerAppendInfo))  
  
            // update the transaction index with the true last stable offset. The last offset visible  
            // to consumers using READ_COMMITTED will be limited by this value and the high watermark.            completedTxns.foreach { completedTxn =&gt;  
              val lastStableOffset = producerStateManager.lastStableOffset(completedTxn)  
              segment.updateTxnIndex(completedTxn, lastStableOffset)  
              producerStateManager.completeTxn(completedTxn)  
            }  
  
            // always update the last producer id map offset so that the snapshot reflects the current offset  
            // even if there isn&#39;t any idempotent data being written            producerStateManager.updateMapEndOffset(appendInfo.lastOffset &#43; 1)  
  
            // update the first unstable offset (which is used to compute LSO)  
            maybeIncrementFirstUnstableOffset()  
  
            trace(s&#34;Appended message set with last offset: ${appendInfo.lastOffset}, &#34; &#43;  
              s&#34;first offset: ${appendInfo.firstOffset}, &#34; &#43;  
              s&#34;next offset: ${localLog.logEndOffset}, &#34; &#43;  
              s&#34;and messages: $validRecords&#34;)  
  
            if (localLog.unflushedMessages &gt;= config.flushInterval) flush(false)  
        }  
        appendInfo  
      }  
    }  
  }  
}
```

```scala
private[log] def append(lastOffset: Long, largestTimestamp: Long, shallowOffsetOfMaxTimestamp: Long, records: MemoryRecords): Unit = {  
  segments.activeSegment.append(lastOffset, largestTimestamp, shallowOffsetOfMaxTimestamp, records)  
  updateLogEndOffset(lastOffset &#43; 1)  
}
```


LogSegment写入流程：

```java
/**  
 * @param largestOffset 最大消息位移值
 * @param largestTimestampMs 最大消息时间戳 
 * @param shallowOffsetOfMaxTimestamp 最大时间戳对应的消息位移
 * @param records 消息体
 */
 public void append(long largestOffset,  
                   long largestTimestampMs,  
                   long shallowOffsetOfMaxTimestamp,  
                   MemoryRecords records) throws IOException {  
    //1. 判断消息是否为空  
    if (records.sizeInBytes() &gt; 0) {  
        LOGGER.trace(&#34;Inserting {} bytes at end offset {} at position {} with largest timestamp {} at offset {}&#34;,  
            records.sizeInBytes(), largestOffset, log.sizeInBytes(), largestTimestampMs, shallowOffsetOfMaxTimestamp);  
        int physicalPosition = log.sizeInBytes();  
        if (physicalPosition == 0)  
            //1.1 如果日志体为空  
            rollingBasedTimestamp = OptionalLong.of(largestTimestampMs);  
  
        //2. 确保offset合法，规则：0 &lt;= largestOffset - baseOffset &lt;= Integer.MAX_VALUE  
        ensureOffsetInRange(largestOffset);  
  
        // 3. 调用FileRecords的append进行写入  
        long appendedBytes = log.append(records);  
        LOGGER.trace(&#34;Appended {} to {} at end offset {}&#34;, appendedBytes, log.file(), largestOffset);  
        // 4. 更新日志的最大时间戳  
        if (largestTimestampMs &gt; maxTimestampSoFar()) {  
            maxTimestampAndOffsetSoFar = new TimestampOffset(largestTimestampMs, shallowOffsetOfMaxTimestamp);  
        }  
        // 4.1 判断是否需要更新索引  
		if (bytesSinceLastIndexEntry &gt; indexIntervalBytes) {  
		    //4.1.1 更新offset index  
		    offsetIndex().append(largestOffset, physicalPosition);  
		    //4.1.2 更新time index  
		    timeIndex().maybeAppend(maxTimestampSoFar(), shallowOffsetOfMaxTimestampSoFar());  
		    //4.1.3 重置bytesSinceLastIndexEntry  
		    bytesSinceLastIndexEntry = 0;  
		} 
        //4.2 更新bytesSinceLastIndexEntry  
        bytesSinceLastIndexEntry &#43;= records.sizeInBytes();  
    }  
}
```


## maybeAddDelayedProduce

向远程其他副本发送写入请求需要满足三个条件：  
 1. acks = -1 
 2. 写入分区消息非空
 3. 本地leader副本的分区消息，至少有一条成功写入 

```scala
private def maybeAddDelayedProduce(  
  requiredAcks: Short,  
  delayedProduceLock: Option[Lock],  
  timeoutMs: Long,  
  entriesPerPartition: Map[TopicPartition, MemoryRecords],  
  initialAppendResults: Map[TopicPartition, LogAppendResult],  
  initialProduceStatus: Map[TopicPartition, ProducePartitionStatus],  
  responseCallback: Map[TopicPartition, PartitionResponse] =&gt; Unit,  
): Unit = {  
  
  //1.1 检查是否满足远程写入的要求  
  if (delayedProduceRequestRequired(requiredAcks, entriesPerPartition, initialAppendResults)) {  
    // create delayed produce operation  
    val produceMetadata = ProduceMetadata(requiredAcks, initialProduceStatus)  
    val delayedProduce = new DelayedProduce(timeoutMs, produceMetadata, this, responseCallback, delayedProduceLock)  
  
    // create a list of (topic, partition) pairs to use as keys for this delayed produce operation  
    val producerRequestKeys = entriesPerPartition.keys.map(TopicPartitionOperationKey(_)).toSeq  
  
    // try to complete the request immediately, otherwise put it into the purgatory  
    // this is because while the delayed produce operation is being created, new    // requests may arrive and hence make this operation completable.    delayedProducePurgatory.tryCompleteElseWatch(delayedProduce, producerRequestKeys)  
  } else {  
    // we can respond immediately  
    val produceResponseStatus = initialProduceStatus.map { case (k, status) =&gt; k -&gt; status.responseStatus }  
    responseCallback(produceResponseStatus)  
  }  
}
```


### read

读取日志信息

```java
/**  
 * Read a message set from this segment that contains startOffset. The message set will include * no more than maxSize bytes and will end before maxOffset if a maxOffset is specified. * * This method is thread-safe. * * @param startOffset 需要读取的位移开始位置  
 * @param maxSize 读取的最大字节数  
 * @param maxPositionOpt 能读取到的最大文件位置  
 * @param minOneMessage 是否允许消息体过大时，返回的第一条消息size &gt; maxSize  
 * * @return The fetched data and the base offset metadata of the message batch that contains startOffset,  
 *         or null if the startOffset is larger than the largest offset in this log */
public FetchDataInfo read(long startOffset, int maxSize, Optional&lt;Long&gt; maxPositionOpt, boolean minOneMessage) throws IOException {  
    if (maxSize &lt; 0)  
        throw new IllegalArgumentException(&#34;Invalid max size &#34; &#43; maxSize &#43; &#34; for log read from segment &#34; &#43; log);  
  
    //1. 调用translateOffset定位到读取的起始文件位置  
    LogOffsetPosition startOffsetAndSize = translateOffset(startOffset);  
  
    // if the start position is already off the end of the log, return null  
    if (startOffsetAndSize == null)  
        return null;  
  
    int startPosition = startOffsetAndSize.position;  
    LogOffsetMetadata offsetMetadata = new LogOffsetMetadata(startOffsetAndSize.offset, this.baseOffset, startPosition);  
  
    //1.2 判断消息最大读取长度  
    int adjustedMaxSize = maxSize;  
    if (minOneMessage)  
        adjustedMaxSize = Math.max(maxSize, startOffsetAndSize.size);  
  
    // return empty records in the fetch-data-info when:  
    // 1. adjustedMaxSize is 0 (or)    // 2. maxPosition to read is unavailable    if (adjustedMaxSize == 0 || !maxPositionOpt.isPresent())  
        return new FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY);  
  
    // calculate the length of the message set to read based on whether or not they gave us a maxOffset  
    int fetchSize = Math.min((int) (maxPositionOpt.get() - startPosition), adjustedMaxSize);  
  
    // 2. 调用log.slice读取数据  
    return new FetchDataInfo(offsetMetadata, log.slice(startPosition, fetchSize),  
        adjustedMaxSize &lt; startOffsetAndSize.size, Optional.empty());  
}
```


# 日志读取


日志Fetch动作由ReplicaManager中fetchMessages()方法来实现。

```scala
def fetchMessages(params: FetchParams,  
                  fetchInfos: Seq[(TopicIdPartition, PartitionData)],  
                  quota: ReplicaQuota,  
                  responseCallback: Seq[(TopicIdPartition, FetchPartitionData)] =&gt; Unit): Unit = {  
  
  //1.1 调用readFromLog方法，获取消息  
  // check if this fetch request can be satisfied right away  
  val logReadResults = readFromLog(params, fetchInfos, quota, readFromPurgatory = false)  
  var bytesReadable: Long = 0  
  var errorReadingData = false  
  
  // The 1st topic-partition that has to be read from remote storage  
  var remoteFetchInfo: Optional[RemoteStorageFetchInfo] = Optional.empty()  
  
  var hasDivergingEpoch = false  
  var hasPreferredReadReplica = false  
  val logReadResultMap = new mutable.HashMap[TopicIdPartition, LogReadResult]  
  
  //2.1 更新统计指标  
  logReadResults.foreach { case (topicIdPartition, logReadResult) =&gt;  
    brokerTopicStats.topicStats(topicIdPartition.topicPartition.topic).totalFetchRequestRate.mark()  
    brokerTopicStats.allTopicsStats.totalFetchRequestRate.mark()  
    if (logReadResult.error != Errors.NONE)  
      errorReadingData = true  
    if (!remoteFetchInfo.isPresent &amp;&amp; logReadResult.info.delayedRemoteStorageFetch.isPresent) {  
      remoteFetchInfo = logReadResult.info.delayedRemoteStorageFetch  
    }  
    if (logReadResult.divergingEpoch.nonEmpty)  
      hasDivergingEpoch = true  
    if (logReadResult.preferredReadReplica.nonEmpty)  
      hasPreferredReadReplica = true  
    bytesReadable = bytesReadable &#43; logReadResult.info.records.sizeInBytes  
    logReadResultMap.put(topicIdPartition, logReadResult)  
  }  
  
  //不需要远程获取，或者满足任一条件  
  // Respond immediately if no remote fetches are required and any of the below conditions is true  
  //                        1) fetch request does not want to wait  //                        2) fetch request does not require any data  //                        3) has enough data to respond  //                        4) some error happens while reading data  //                        5) we found a diverging epoch  //                        6) has a preferred read replica  if (!remoteFetchInfo.isPresent &amp;&amp; (params.maxWaitMs &lt;= 0 || fetchInfos.isEmpty || bytesReadable &gt;= params.minBytes || errorReadingData ||  
    hasDivergingEpoch || hasPreferredReadReplica)) {  
    val fetchPartitionData = logReadResults.map { case (tp, result) =&gt;  
      val isReassignmentFetch = params.isFromFollower &amp;&amp; isAddingReplica(tp.topicPartition, params.replicaId)  
      tp -&gt; result.toFetchPartitionData(isReassignmentFetch)  
    }  
    responseCallback(fetchPartitionData)  
  } else {  
    //构建返回结果  
    // construct the fetch results from the read results  
    val fetchPartitionStatus = new mutable.ArrayBuffer[(TopicIdPartition, FetchPartitionStatus)]  
    fetchInfos.foreach { case (topicIdPartition, partitionData) =&gt;  
      logReadResultMap.get(topicIdPartition).foreach(logReadResult =&gt; {  
        val logOffsetMetadata = logReadResult.info.fetchOffsetMetadata  
        fetchPartitionStatus &#43;= (topicIdPartition -&gt; FetchPartitionStatus(logOffsetMetadata, partitionData))  
      })  
    }  
  
    //执行远程fetch  
    if (remoteFetchInfo.isPresent) {  
      val maybeLogReadResultWithError = processRemoteFetch(remoteFetchInfo.get(), params, responseCallback, logReadResults, fetchPartitionStatus)  
      if (maybeLogReadResultWithError.isDefined) {  
        // If there is an error in scheduling the remote fetch task, return what we currently have  
        // (the data read from local log segment for the other topic-partitions) and an error for the topic-partition        // that we couldn&#39;t read from remote storage        val partitionToFetchPartitionData = buildPartitionToFetchPartitionData(logReadResults, remoteFetchInfo.get().topicPartition, maybeLogReadResultWithError.get)  
        responseCallback(partitionToFetchPartitionData)  
      }  
    } else {  
      // If there is not enough data to respond and there is no remote data, we will let the fetch request  
      // wait for new data.      val delayedFetch = new DelayedFetch(  
        params = params,  
        fetchPartitionStatus = fetchPartitionStatus,  
        replicaManager = this,  
        quota = quota,  
        responseCallback = responseCallback  
      )  
  
      // create a list of (topic, partition) pairs to use as keys for this delayed fetch operation  
      val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) =&gt; TopicPartitionOperationKey(tp) }  
  
      // try to complete the request immediately, otherwise put it into the purgatory;  
      // this is because while the delayed fetch operation is being created, new requests      // may arrive and hence make this operation completable.      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)  
    }  
  }  
}s
```


readFromLog()方法中通过partition读取消息：
```scala
//调用partition的fetchRecords读取消息  
// Try the read first, this tells us whether we need all of adjustedFetchSize for this partition  
val readInfo: LogReadInfo = partition.fetchRecords(  
  fetchParams = params,  
  fetchPartitionData = fetchInfo,  
  fetchTimeMs = fetchTimeMs,  
  maxBytes = adjustedMaxBytes,  
  minOneMessage = minOneMessage,  
  updateFetchState = !readFromPurgatory)
```


```scala
def fetchRecords(  
  fetchParams: FetchParams,  
  fetchPartitionData: FetchRequest.PartitionData,  
  fetchTimeMs: Long,  
  maxBytes: Int,  
  minOneMessage: Boolean,  
  updateFetchState: Boolean  
): LogReadInfo = {  
  
  //定义  
  def readFromLocalLog(log: UnifiedLog): LogReadInfo = {  
    readRecords(  
      log,  
      fetchPartitionData.lastFetchedEpoch,  
      fetchPartitionData.fetchOffset,  
      fetchPartitionData.currentLeaderEpoch,  
      maxBytes,  
      fetchParams.isolation,  
      minOneMessage  
    )  
  }  
  
  //检查请求是否来自partition的follower  
  if (fetchParams.isFromFollower) {  
    //需要检查是否来自有效的follower  
    // Check that the request is from a valid replica before doing the read    val (replica, logReadInfo) = inReadLock(leaderIsrUpdateLock) {  
      val localLog = localLogWithEpochOrThrow(  
        fetchPartitionData.currentLeaderEpoch,  
        fetchParams.fetchOnlyLeader  
      )  
      val replica = followerReplicaOrThrow(  
        fetchParams.replicaId,  
        fetchPartitionData  
      )  
      val logReadInfo = readFromLocalLog(localLog)  
      (replica, logReadInfo)  
    }  
  
    if (updateFetchState &amp;&amp; !logReadInfo.divergingEpoch.isPresent) {  
      updateFollowerFetchState(  
        replica,  
        followerFetchOffsetMetadata = logReadInfo.fetchedData.fetchOffsetMetadata,  
        followerStartOffset = fetchPartitionData.logStartOffset,  
        followerFetchTimeMs = fetchTimeMs,  
        leaderEndOffset = logReadInfo.logEndOffset,  
        fetchParams.replicaEpoch  
      )  
    }  
  
    logReadInfo  
  } else {  
    inReadLock(leaderIsrUpdateLock) {  
      val localLog = localLogWithEpochOrThrow(  
        fetchPartitionData.currentLeaderEpoch,  
        fetchParams.fetchOnlyLeader  
      )  
      readFromLocalLog(localLog)  
    }  
  }  
}
```


kafka.log.UnifiedLog#read：
```scala
def read(startOffset: Long,  
         maxLength: Int,  
         isolation: FetchIsolation,  
         minOneMessage: Boolean): FetchDataInfo = {  
  checkLogStartOffset(startOffset)  
  val maxOffsetMetadata = isolation match {  
    case FetchIsolation.LOG_END =&gt; localLog.logEndOffsetMetadata  
    case FetchIsolation.HIGH_WATERMARK =&gt; fetchHighWatermarkMetadata  
    case FetchIsolation.TXN_COMMITTED =&gt; fetchLastStableOffsetMetadata  
  }  
  localLog.read(startOffset, maxLength, minOneMessage, maxOffsetMetadata, isolation == FetchIsolation.TXN_COMMITTED)  
}
```


kafka.log.LocalLog#read：
```scala
def read(startOffset: Long,  
         maxLength: Int,  
         minOneMessage: Boolean,  
         maxOffsetMetadata: LogOffsetMetadata,  
         includeAbortedTxns: Boolean): FetchDataInfo = {  
  maybeHandleIOException(s&#34;Exception while reading from $topicPartition in dir ${dir.getParent}&#34;) {  
    trace(s&#34;Reading maximum $maxLength bytes at offset $startOffset from log with &#34; &#43;  
      s&#34;total length ${segments.sizeInBytes} bytes&#34;)  
  
  
    //1.1 选择offset所在的segment  
    val endOffsetMetadata = nextOffsetMetadata  
    val endOffset = endOffsetMetadata.messageOffset  
    var segmentOpt = segments.floorSegment(startOffset)  
  
    // return error on attempt to read beyond the log end offset  
    if (startOffset &gt; endOffset || !segmentOpt.isPresent)  
      throw new OffsetOutOfRangeException(s&#34;Received request for offset $startOffset for partition $topicPartition, &#34; &#43;  
        s&#34;but we only have log segments upto $endOffset.&#34;)  
  
    if (startOffset == maxOffsetMetadata.messageOffset)  
      emptyFetchDataInfo(maxOffsetMetadata, includeAbortedTxns)  
    else if (startOffset &gt; maxOffsetMetadata.messageOffset)  
      emptyFetchDataInfo(convertToOffsetMetadataOrThrow(startOffset), includeAbortedTxns)  
    else {  
      // Do the read on the segment with a base offset less than the target offset  
      // but if that segment doesn&#39;t contain any messages with an offset greater than that      // continue to read from successive segments until we get some messages or we reach the end of the log      var fetchDataInfo: FetchDataInfo = null  
      while (fetchDataInfo == null &amp;&amp; segmentOpt.isPresent) {  
        val segment = segmentOpt.get  
        val baseOffset = segment.baseOffset  
  
        // 1. If `maxOffsetMetadata#segmentBaseOffset &lt; segment#baseOffset`, then return maxPosition as empty.  
        // 2. Use the max-offset position if it is on this segment; otherwise, the segment size is the limit.        // 3. When maxOffsetMetadata is message-offset-only, then we don&#39;t know the relativePositionInSegment so        //    return maxPosition as empty to avoid reading beyond the max-offset        val maxPositionOpt: Optional[java.lang.Long] =  
          if (segment.baseOffset &lt; maxOffsetMetadata.segmentBaseOffset)  
            Optional.of(segment.size)  
          else if (segment.baseOffset == maxOffsetMetadata.segmentBaseOffset &amp;&amp; !maxOffsetMetadata.messageOffsetOnly())  
            Optional.of(maxOffsetMetadata.relativePositionInSegment)  
          else  
            Optional.empty()  
  
        fetchDataInfo = segment.read(startOffset, maxLength, maxPositionOpt, minOneMessage)  
        if (fetchDataInfo != null) {  
          if (includeAbortedTxns)  
            fetchDataInfo = addAbortedTransactions(startOffset, segment, fetchDataInfo)  
        } else segmentOpt = segments.higherSegment(baseOffset)  
      }  
  
      if (fetchDataInfo != null) fetchDataInfo  
      else {  
        // okay we are beyond the end of the last segment with no data fetched although the start offset is in range,  
        // this can happen when all messages with offset larger than start offsets have been deleted.        // In this case, we will return the empty set with log end offset metadata        new FetchDataInfo(nextOffsetMetadata, MemoryRecords.EMPTY)  
      }  
    }  
  }  
}
```


通过LogSegment读取指定位移的消息：

```java
public FetchDataInfo read(long startOffset, int maxSize, Optional&lt;Long&gt; maxPositionOpt, boolean minOneMessage) throws IOException {  
    if (maxSize &lt; 0)  
        throw new IllegalArgumentException(&#34;Invalid max size &#34; &#43; maxSize &#43; &#34; for log read from segment &#34; &#43; log);  
  
    //1.1 将startOffset转换为物理文件位置  
    LogOffsetPosition startOffsetAndSize = translateOffset(startOffset);  
  
    // if the start position is already off the end of the log, return null  
    if (startOffsetAndSize == null)  
        return null;  
  
    int startPosition = startOffsetAndSize.position;  
    LogOffsetMetadata offsetMetadata = new LogOffsetMetadata(startOffset, this.baseOffset, startPosition);  
  
    int adjustedMaxSize = maxSize;  
    if (minOneMessage)  
        adjustedMaxSize = Math.max(maxSize, startOffsetAndSize.size);  
  
    // return empty records in the fetch-data-info when:  
    // 1. adjustedMaxSize is 0 (or)    // 2. maxPosition to read is unavailable    if (adjustedMaxSize == 0 || !maxPositionOpt.isPresent())  
        return new FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY);  
  
    // calculate the length of the message set to read based on whether or not they gave us a maxOffset  
    int fetchSize = Math.min((int) (maxPositionOpt.get() - startPosition), adjustedMaxSize);  
  
    //调用slice获取消息  
    return new FetchDataInfo(offsetMetadata, log.slice(startPosition, fetchSize),  
        adjustedMaxSize &lt; startOffsetAndSize.size, Optional.empty());  
}
```


这里并没有从本地文件中读取实际的消息，这是lazy-load方式，实际读取逻辑在：org.apache.kafka.common.record.FileLogInputStream.FileChannelRecordBatch#loadBatchWithSize

```java
private RecordBatch loadBatchWithSize(int size, String description) {  
    FileChannel channel = fileRecords.channel();  
    try {  
        ByteBuffer buffer = ByteBuffer.allocate(size);  
        Utils.readFullyOrFail(channel, buffer, position, description);  
        buffer.rewind();  
        return toMemoryRecordBatch(buffer);  
    } catch (IOException e) {  
        throw new KafkaException(&#34;Failed to load record batch at position &#34; &#43; position &#43; &#34; from &#34; &#43; fileRecords, e);  
    }  
}s
```


# 索引重建

broker启动加载所有segment时，若Index文件损坏或缺失，会调用索引重建逻辑：
```scala
try segment.sanityCheck(timeIndexFileNewlyCreated)  
catch {  
  case _: NoSuchFileException =&gt;  
    if (hadCleanShutdown || segment.baseOffset &lt; recoveryPointCheckpoint)  
      error(s&#34;Could not find offset index file corresponding to log file&#34; &#43;  
        s&#34; ${segment.log.file.getAbsolutePath}, recovering segment and rebuilding index files...&#34;)  
    recoverSegment(segment)  
  case e: CorruptIndexException =&gt;  
    warn(s&#34;Found a corrupted index file corresponding to log file&#34; &#43;  
      s&#34; ${segment.log.file.getAbsolutePath} due to ${e.getMessage}}, recovering segment and&#34; &#43;  
      &#34; rebuilding index files...&#34;)  
    recoverSegment(segment)  
}
```

```scala
private def recoverSegment(segment: LogSegment): Int = {  
  val producerStateManager = new ProducerStateManager(  
    topicPartition,  
    dir,  
    this.producerStateManager.maxTransactionTimeoutMs(),  
    this.producerStateManager.producerStateManagerConfig(),  
    time)  
  UnifiedLog.rebuildProducerState(  
    producerStateManager,  
    segments,  
    logStartOffsetCheckpoint,  
    segment.baseOffset,  
    config.recordVersion,  
    time,  
    reloadFromCleanShutdown = false,  
    logIdent)  
  val bytesTruncated = segment.recover(producerStateManager, leaderEpochCache)  
  // once we have recovered the segment&#39;s data, take a snapshot to ensure that we won&#39;t  
  // need to reload the same segment again while recovering another segment.  producerStateManager.takeSnapshot()  
  bytesTruncated  
}
```


```java
/**  
 * Run recovery on the given segment. This will rebuild the index from the log file and lop off any invalid bytes * from the end of the log and index. * * This method is not thread-safe. * * @param producerStateManager producer状态管理，用于恢复事务索引  
 * @param leaderEpochCache 用于更新leader选举周期  
 * @return 从log中截断的字节数  
 * @throws LogSegmentOffsetOverflowException if the log segment contains an offset that causes the index offset to overflow  
 */
 public int recover(ProducerStateManager producerStateManager, Optional&lt;LeaderEpochFileCache&gt; leaderEpochCache) throws IOException {  
  
    // 1. 清空所有的索引文件  
    offsetIndex().reset();  
    timeIndex().reset();  
    txnIndex.reset();  
    int validBytes = 0;  
    int lastIndexEntry = 0;  
    maxTimestampAndOffsetSoFar = TimestampOffset.UNKNOWN;  
    try {  
        //2. 遍历log的RecordBatch  
        for (RecordBatch batch : log.batches()) {  
            //2.1 校验batch是否合法  
            batch.ensureValid();  
  
            //2.2 保证最后一条消息offset在范围内  
            ensureOffsetInRange(batch.lastOffset());  
  
            //2.3 校验batch最大时间戳是否大于segment最大时间戳  
            if (batch.maxTimestamp() &gt; maxTimestampSoFar()) {  
                //2.3.1 更新maxTimestampAndOffsetSoFar  
                maxTimestampAndOffsetSoFar = new TimestampOffset(batch.maxTimestamp(), batch.lastOffset());  
            }  
  
            // 3. 构建index  
            if (validBytes - lastIndexEntry &gt; indexIntervalBytes) {  
                offsetIndex().append(batch.lastOffset(), validBytes);  
                timeIndex().maybeAppend(maxTimestampSoFar(), shallowOffsetOfMaxTimestampSoFar());  
                lastIndexEntry = validBytes;  
            }  
  
            //3.1 更新validBytes  
            validBytes &#43;= batch.sizeInBytes();  
  
            //4. 更新leader选举周期  
            if (batch.magic() &gt;= RecordBatch.MAGIC_VALUE_V2) {  
                leaderEpochCache.ifPresent(cache -&gt; {  
                    // 4.1 获取batch的选举周期，如果 partitionLeaderEpoch &gt; cache latestEpoch，则更新cache  
                    if (batch.partitionLeaderEpoch() &gt;= 0 &amp;&amp;  
                            (!cache.latestEpoch().isPresent() || batch.partitionLeaderEpoch() &gt; cache.latestEpoch().getAsInt()))  
                        cache.assign(batch.partitionLeaderEpoch(), batch.baseOffset());  
                });  
                //4.2 更新producer状态  
                updateProducerState(producerStateManager, batch);  
            }  
        }  
    } catch (CorruptRecordException | InvalidRecordException e) {  
        LOGGER.warn(&#34;Found invalid messages in log segment {} at byte offset {}.&#34;, log.file().getAbsolutePath(),  
            validBytes, e);  
    }  
    int truncated = log.sizeInBytes() - validBytes;  
    if (truncated &gt; 0)  
        LOGGER.debug(&#34;Truncated {} invalid bytes at the end of segment {} during recovery&#34;, truncated, log.file().getAbsolutePath());  
    //5. 底层调用FileChannel的truncate方法截断文件，用于清理末尾的无效字节
    log.truncateTo(validBytes);  
    offsetIndex().trimToValidSize();  
    // A normally closed segment always appends the biggest timestamp ever seen into log segment, we do this as well.  
    timeIndex().maybeAppend(maxTimestampSoFar(), shallowOffsetOfMaxTimestampSoFar(), true); 
    timeIndex().trimToValidSize();  
    return truncated;  
}
```

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-4-log-rw/  

