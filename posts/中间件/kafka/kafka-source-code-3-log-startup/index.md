# Kafka源码分析(三) - 日志加载流程


Kafka Log由多个LogSegment构成，每个LogSegment对应一个分区，每个LogSegment对象都会在磁盘中创建一组文件：
1. 日志消息文件(.log)
2. 位移索引文件(.index)
3. 时间戳索引文件(.timeindex)
4. 已中止十五的索引文件(.txnindex)


## LogManager

LogManager作为Kafka Log管理的入口点，负责日志创建、检索、清理，所有的读写操作都会委托给对应的日志实例。

```scala
private val currentLogs = new Pool[TopicPartition, UnifiedLog]()
private val _liveLogDirs: ConcurrentLinkedQueue[File] = createAndValidateLogDirs(logDirs, initialOfflineDirs)
```

currentLogs字段是用于存储分区和UnifiedLog映射的map结构，Pool内部使用ConcurrentHashMap作为存储。

liveLogDirs用于存储log路径下所有有效的File对象，存储结构使用ConcurrentLinkedQueue。createAndValidateLogDirs()方法用来检查并追加有效路径到队列中。


### startup启动流程

KafkaServer的startup()方法中会调用logManager的startup()方法，用来加载所有的日志。
```scala
logManager.startup(zkClient.getAllTopicsInCluster())
```


startup的第一个参数是topic名称集合，第二个可选参数isStray通过metadata计算当前UnifiedLog是否需要当前broker加载（kafka.log.LogManager.isStrayKraftReplica）

```scala
/**  
 *  Start the background threads to flush logs and do log cleanup */def startup(topicNames: Set[String], isStray: UnifiedLog =&gt; Boolean = _ =&gt; false): Unit = {  
  // ensure consistency between default config and overrides  
  val defaultConfig = currentDefaultConfig  
  startupWithConfigOverrides(defaultConfig, fetchTopicConfigOverrides(defaultConfig, topicNames), isStray)  
}s
```

fetchTopicConfigOverrides()方法用于检查topic是否存在自定义配置并覆盖。接下来看startupWithConfigOverrides()的具体实现。

1. 调用loadLogs()加载所有log
2. 启动以下定时调度任务：
	1. 日志清理任务
	2. 日志刷盘任务
	3. 日志checkpoint更新任务
	4. log start offset属性checkpoint更新任务
	5. 日志删除任务
3. 检查`log.cleaner.enable`参数，true则配置LogCleaner


### scheduler定时调度

scheduler定时调度使用内部封装的`org.apache.kafka.server.util.KafkaScheduler`，它是对ScheduledThreadPoolExecutor的简单封装。

```scala
private[log] def startupWithConfigOverrides(  
  defaultConfig: LogConfig,  
  topicConfigOverrides: Map[String, LogConfig],  
  isStray: UnifiedLog =&gt; Boolean): Unit = {  
    
  loadLogs(defaultConfig, topicConfigOverrides, isStray) // this could take a while if shutdown was not clean  
  
  /* Schedule the cleanup task to delete old logs */  if (scheduler != null) {  
    info(&#34;Starting log cleanup with a period of %d ms.&#34;.format(retentionCheckMs))  
    scheduler.schedule(&#34;kafka-log-retention&#34;,  
                       () =&gt; cleanupLogs(),  
                       initialTaskDelayMs,  
                       retentionCheckMs)  
    info(&#34;Starting log flusher with a default period of %d ms.&#34;.format(flushCheckMs))  
    scheduler.schedule(&#34;kafka-log-flusher&#34;,  
                       () =&gt; flushDirtyLogs(),  
                       initialTaskDelayMs,  
                       flushCheckMs)  
    scheduler.schedule(&#34;kafka-recovery-point-checkpoint&#34;,  
                       () =&gt; checkpointLogRecoveryOffsets(),  
                       initialTaskDelayMs,  
                       flushRecoveryOffsetCheckpointMs)  
    scheduler.schedule(&#34;kafka-log-start-offset-checkpoint&#34;,  
                       () =&gt; checkpointLogStartOffsets(),  
                       initialTaskDelayMs,  
                       flushStartOffsetCheckpointMs)  
    scheduler.scheduleOnce(&#34;kafka-delete-logs&#34;, // will be rescheduled after each delete logs with a dynamic period  
                       () =&gt; deleteLogs(),  
                       initialTaskDelayMs)  
  }  
  if (cleanerConfig.enableCleaner) {  
    _cleaner = new LogCleaner(cleanerConfig, liveLogDirs, currentLogs, logDirFailureChannel, time = time)  
    _cleaner.startup()  
  }  
}
```


### loadLogs流程

loadLogs()方法作为实际加载Log数据的入口，逻辑链路较长，首先看下方法中使用到的类属性：

```scala
// A map that stores hadCleanShutdown flag for each log dir.  
private val hadCleanShutdownFlags = new ConcurrentHashMap[String, Boolean]()  
  
// A map that tells whether all logs in a log dir had been loaded or not at startup time.  
private val loadLogsCompletedFlags = new ConcurrentHashMap[String, Boolean]()

@volatile private var recoveryPointCheckpoints = liveLogDirs.map(dir =&gt;  
  (dir, new OffsetCheckpointFile(new File(dir, RecoveryPointCheckpointFile), logDirFailureChannel))).toMap  


@volatile private var logStartOffsetCheckpoints = liveLogDirs.map(dir =&gt;  
  (dir, new OffsetCheckpointFile(new File(dir, LogStartOffsetCheckpointFile), logDirFailureChannel))).toMap
```

其中hadCleanShutdownFlags和loadLogsCompletedFlags都是`map[String, Boolean]` 结构，分别用于存储对应log dir是否为clean shutdown、是否已经完成加载的flag。

recoveryPointCheckpoints和logStartOffsetCheckpoints是`map[String, OffsetCheckpointFile]`结构，OffsetCheckpointFile对象是`Partition =&gt; Offsets`的映射持久化存储的相关对象。类似的文件格式如下：
```
-----checkpoint file begin------  
0                &lt;- OffsetCheckpointFile.currentVersion  
2                &lt;- following entries size  
tp1  par1  1     &lt;- the format is: TOPIC  PARTITION  OFFSET  
tp1  par2  2  
-----checkpoint file end----------
```


loadLogs()方法的核心逻辑如下：
1. 循环当前data dirs，每个data dir分配一个线程池执行load
2. 获取当前dir下topic的recovery point和log start offset checkpoint信息
3. 遍历当前dir下所有topic，忽略掉remote存储的，构建runnable加载任务，runnable任务会调用loadLog构造UnifiedLog

```scala
/**  
 * Recover and load all logs in the given data directories */private[log] def loadLogs(defaultConfig: LogConfig, topicConfigOverrides: Map[String, LogConfig], isStray: UnifiedLog =&gt; Boolean): Unit = {  
  info(s&#34;Loading logs from log dirs $liveLogDirs&#34;)  
  
  //1.1 定义线程池和任务队列  
  val startMs = time.hiResClockMs()  
  val threadPools = ArrayBuffer.empty[ExecutorService]  
  val offlineDirs = mutable.Set.empty[(String, IOException)]  
  val jobs = ArrayBuffer.empty[Seq[Future[_]]]  
  var numTotalLogs = 0  
  
  // log dir path -&gt; number of Remaining logs map for remainingLogsToRecover metric  
  val numRemainingLogs: ConcurrentMap[String, Int] = new ConcurrentHashMap[String, Int]  
  // log recovery thread name -&gt; number of remaining segments map for remainingSegmentsToRecover metric  
  val numRemainingSegments: ConcurrentMap[String, Int] = new ConcurrentHashMap[String, Int]  
  
  def handleIOException(logDirAbsolutePath: String, e: IOException): Unit = {  
    offlineDirs.add((logDirAbsolutePath, e))  
    error(s&#34;Error while loading log dir $logDirAbsolutePath&#34;, e)  
  }  
  
  //1.2 循环liveLogDirs  
  val uncleanLogDirs = mutable.Buffer.empty[String]  
  for (dir &lt;- liveLogDirs) {  
    val logDirAbsolutePath = dir.getAbsolutePath  
    var hadCleanShutdown: Boolean = false  
    //1.3 每个data dir分配一个线程池去加载  
    try {  
      val pool = Executors.newFixedThreadPool(numRecoveryThreadsPerDataDir,  
        new LogRecoveryThreadFactory(logDirAbsolutePath))  
      threadPools.append(pool)  
  
      //1.4 检查上次是否为clean shutdown, 删除文件是为了防止加载日志时发生crash后仍然被认为是clean shutdown  
      val cleanShutdownFileHandler = new CleanShutdownFileHandler(dir.getPath)  
      if (cleanShutdownFileHandler.exists()) {  
        // Cache the clean shutdown status and use that for rest of log loading workflow. Delete the CleanShutdownFile  
        // so that if broker crashes while loading the log, it is considered hard shutdown during the next boot up. KAFKA-10471        cleanShutdownFileHandler.delete()  
        hadCleanShutdown = true  
      }  
  
      //1.5 更新当前log dir是否为clean shutdown  
      hadCleanShutdownFlags.put(logDirAbsolutePath, hadCleanShutdown)  
  
  
      //1.6 获取当前dir的recovery point信息和log start offset信息  
      var recoveryPoints = Map[TopicPartition, Long]()  
      try {  
        recoveryPoints = this.recoveryPointCheckpoints(dir).read()  
      } catch {  
        case e: Exception =&gt;  
          warn(s&#34;Error occurred while reading recovery-point-offset-checkpoint file of directory &#34; &#43;  
            s&#34;$logDirAbsolutePath, resetting the recovery checkpoint to 0&#34;, e)  
      }  
  
      var logStartOffsets = Map[TopicPartition, Long]()  
      try {  
        logStartOffsets = this.logStartOffsetCheckpoints(dir).read()  
      } catch {  
        case e: Exception =&gt;  
          warn(s&#34;Error occurred while reading log-start-offset-checkpoint file of directory &#34; &#43;  
            s&#34;$logDirAbsolutePath, resetting to the base offset of the first segment&#34;, e)  
      }  
  
      //1.7 忽略remote存储的topic  
      val logsToLoad = Option(dir.listFiles).getOrElse(Array.empty).filter(logDir =&gt;  
        logDir.isDirectory &amp;&amp;  
          // Ignore remote-log-index-cache directory as that is index cache maintained by tiered storage subsystem  
          // but not any topic-partition dir.          !logDir.getName.equals(RemoteIndexCache.DIR_NAME) &amp;&amp;  
          UnifiedLog.parseTopicPartitionName(logDir).topic != KafkaRaftServer.MetadataTopic)  
  
      numTotalLogs &#43;= logsToLoad.length  
      numRemainingLogs.put(logDirAbsolutePath, logsToLoad.length)  
      loadLogsCompletedFlags.put(logDirAbsolutePath, logsToLoad.isEmpty)  
  
      if (logsToLoad.isEmpty) {  
        info(s&#34;No logs found to be loaded in $logDirAbsolutePath&#34;)  
      } else if (hadCleanShutdown) {  
        info(s&#34;Skipping recovery of ${logsToLoad.length} logs from $logDirAbsolutePath since &#34; &#43;  
          &#34;clean shutdown file was found&#34;)  
      } else {  
        info(s&#34;Recovering ${logsToLoad.length} logs from $logDirAbsolutePath since no &#34; &#43;  
          &#34;clean shutdown file was found&#34;)  
        uncleanLogDirs.append(logDirAbsolutePath)  
      }  
  
      //1.8 遍历当前dir，构建runnable加载任务，runnable调用loadLog构造UnifiedLog  
      val jobsForDir = logsToLoad.map { logDir =&gt;  
        val runnable: Runnable = () =&gt; {  
          debug(s&#34;Loading log $logDir&#34;)  
          var log = None: Option[UnifiedLog]  
          val logLoadStartMs = time.hiResClockMs()  
          try {  
            log = Some(loadLog(logDir, hadCleanShutdown, recoveryPoints, logStartOffsets,  
              defaultConfig, topicConfigOverrides, numRemainingSegments, isStray))  
          } catch {  
            case e: IOException =&gt;  
              handleIOException(logDirAbsolutePath, e)  
            case e: KafkaStorageException if e.getCause.isInstanceOf[IOException] =&gt;  
              // KafkaStorageException might be thrown, ex: during writing LeaderEpochFileCache  
              // And while converting IOException to KafkaStorageException, we&#39;ve already handled the exception. So we can ignore it here.          } finally {  
            val logLoadDurationMs = time.hiResClockMs() - logLoadStartMs  
            val remainingLogs = decNumRemainingLogs(numRemainingLogs, logDirAbsolutePath)  
            val currentNumLoaded = logsToLoad.length - remainingLogs  
            log match {  
              case Some(loadedLog) =&gt; info(s&#34;Completed load of $loadedLog with ${loadedLog.numberOfSegments} segments, &#34; &#43;  
                s&#34;local-log-start-offset ${loadedLog.localLogStartOffset()} and log-end-offset ${loadedLog.logEndOffset} in ${logLoadDurationMs}ms &#34; &#43;  
                s&#34;($currentNumLoaded/${logsToLoad.length} completed in $logDirAbsolutePath)&#34;)  
              case None =&gt; info(s&#34;Error while loading logs in $logDir in ${logLoadDurationMs}ms ($currentNumLoaded/${logsToLoad.length} completed in $logDirAbsolutePath)&#34;)  
            }  
  
            if (remainingLogs == 0) {  
              // loadLog is completed for all logs under the logDdir, mark it.  
              loadLogsCompletedFlags.put(logDirAbsolutePath, true)  
            }  
          }  
        }  
        runnable  
      }  
  
      jobs &#43;= jobsForDir.map(pool.submit)  
    } catch {  
      case e: IOException =&gt;  
        handleIOException(logDirAbsolutePath, e)  
    }  
  }  
  
  //1.9 添加Metrics  
  try {  
    addLogRecoveryMetrics(numRemainingLogs, numRemainingSegments)  
    for (dirJobs &lt;- jobs) {  
      dirJobs.foreach(_.get)  
    }  
  
    offlineDirs.foreach { case (dir, e) =&gt;  
      logDirFailureChannel.maybeAddOfflineLogDir(dir, s&#34;Error while loading log dir $dir&#34;, e)  
    }  
  } catch {  
    case e: ExecutionException =&gt;  
      error(s&#34;There was an error in one of the threads during logs loading: ${e.getCause}&#34;)  
      throw e.getCause  
  } finally {  
    removeLogRecoveryMetrics()  
    threadPools.foreach(_.shutdown())  
  }  
  
  val elapsedMs = time.hiResClockMs() - startMs  
  val printedUncleanLogDirs = if (uncleanLogDirs.isEmpty) &#34;&#34; else s&#34; (unclean log dirs = $uncleanLogDirs)&#34;  
  info(s&#34;Loaded $numTotalLogs logs in ${elapsedMs}ms$printedUncleanLogDirs&#34;)  
}
```


loadLog()方法会先创建UnifiedLog对象，在其初始化逻辑中，会调用LogLoader.load()方法完成日志加载动作，UnifiedLog初始化完成后会将该对象存储到map中。

```scala
private[log] def loadLog(logDir: File,  
                         hadCleanShutdown: Boolean,  
                         recoveryPoints: Map[TopicPartition, Long],  
                         logStartOffsets: Map[TopicPartition, Long],  
                         defaultConfig: LogConfig,  
                         topicConfigOverrides: Map[String, LogConfig],  
                         numRemainingSegments: ConcurrentMap[String, Int],  
                         isStray: UnifiedLog =&gt; Boolean): UnifiedLog = {  
  val topicPartition = UnifiedLog.parseTopicPartitionName(logDir)  
  val config = topicConfigOverrides.getOrElse(topicPartition.topic, defaultConfig)  
  val logRecoveryPoint = recoveryPoints.getOrElse(topicPartition, 0L)  
  val logStartOffset = logStartOffsets.getOrElse(topicPartition, 0L)  
  
  
  //1.1 创建UnifiedLog对象，这里会调用UnifiedLog的apply方法  
  val log = UnifiedLog(  
    dir = logDir,  
    config = config,  
    logStartOffset = logStartOffset,  
    recoveryPoint = logRecoveryPoint,  
    maxTransactionTimeoutMs = maxTransactionTimeoutMs,  
    producerStateManagerConfig = producerStateManagerConfig,  
    producerIdExpirationCheckIntervalMs = producerIdExpirationCheckIntervalMs,  
    scheduler = scheduler,  
    time = time,  
    brokerTopicStats = brokerTopicStats,  
    logDirFailureChannel = logDirFailureChannel,  
    lastShutdownClean = hadCleanShutdown,  
    topicId = None,  
    keepPartitionMetadataFile = keepPartitionMetadataFile,  
    numRemainingSegments = numRemainingSegments,  
    remoteStorageSystemEnable = remoteStorageSystemEnable)  
  
  //1.2 检查是否为待删除文件，添加到待删除队列中，后续定时任务执行删除动作  
  if (logDir.getName.endsWith(UnifiedLog.DeleteDirSuffix)) {  
    addLogToBeDeleted(log)  
  } else if (logDir.getName.endsWith(UnifiedLog.StrayDirSuffix)) {  
    //1.2 检查是否为Stray分区  
    addStrayLog(topicPartition, log)  
    warn(s&#34;Loaded stray log: $logDir&#34;)  
  } else if (isStray(log)) {  
    // Unlike Zookeeper mode, which tracks pending topic deletions under a ZNode, KRaft is unable to prevent a topic from being recreated before every replica has been deleted.  
    // A KRaft broker with an offline directory may be unable to detect it still holds a to-be-deleted replica,    // and can create a conflicting topic partition for a new incarnation of the topic in one of the remaining online directories.    // So upon a restart in which the offline directory is back online we need to clean up the old replica directory.    log.renameDir(UnifiedLog.logStrayDirName(log.topicPartition), shouldReinitialize = false)  
    addStrayLog(log.topicPartition, log)  
    warn(s&#34;Log in ${logDir.getAbsolutePath} marked stray and renamed to ${log.dir.getAbsolutePath}&#34;)  
  } else {  
    //previous保存旧值  
    val previous = {  
      if (log.isFuture)  
        this.futureLogs.put(topicPartition, log)  
      else  
        this.currentLogs.put(topicPartition, log)  
    }  
    //check是否重复  
    if (previous != null) {  
      if (log.isFuture)  
        throw new IllegalStateException(s&#34;Duplicate log directories found: ${log.dir.getAbsolutePath}, ${previous.dir.getAbsolutePath}&#34;)  
      else  
        throw new IllegalStateException(s&#34;Duplicate log directories for $topicPartition are found in both ${log.dir.getAbsolutePath} &#34; &#43;  
          s&#34;and ${previous.dir.getAbsolutePath}. It is likely because log directory failure happened while broker was &#34; &#43;  
          s&#34;replacing current replica with future replica. Recover broker from this failure by manually deleting one of the two directories &#34; &#43;  
          s&#34;for this partition. It is recommended to delete the partition in the log directory that is known to have failed recently.&#34;)  
    }  
  }  
  
  log  
}
```

UnifiedLog的apply()方法完成初始化动作：
```scala
def apply(dir: File,  
          config: LogConfig,  
          logStartOffset: Long,  
          recoveryPoint: Long,  
          scheduler: Scheduler,  
          brokerTopicStats: BrokerTopicStats,  
          time: Time,  
          maxTransactionTimeoutMs: Int,  
          producerStateManagerConfig: ProducerStateManagerConfig,  
          producerIdExpirationCheckIntervalMs: Int,  
          logDirFailureChannel: LogDirFailureChannel,  
          lastShutdownClean: Boolean = true,  
          topicId: Option[Uuid],  
          keepPartitionMetadataFile: Boolean,  
          numRemainingSegments: ConcurrentMap[String, Int] = new ConcurrentHashMap[String, Int],  
          remoteStorageSystemEnable: Boolean = false,  
          logOffsetsListener: LogOffsetsListener = LogOffsetsListener.NO_OP_OFFSETS_LISTENER): UnifiedLog = {  
  // create the log directory if it doesn&#39;t exist  
  Files.createDirectories(dir.toPath)  
  
  //1.1 从dir解析出topicPartition，初始化LogSegments  
  val topicPartition = UnifiedLog.parseTopicPartitionName(dir)  
  val segments = new LogSegments(topicPartition)  
  // The created leaderEpochCache will be truncated by LogLoader if necessary  
  // so it is guaranteed that the epoch entries will be correct even when on-disk  // checkpoint was stale (due to async nature of LeaderEpochFileCache#truncateFromStart/End).  val leaderEpochCache = UnifiedLog.maybeCreateLeaderEpochCache(  
    dir,  
    topicPartition,  
    logDirFailureChannel,  
    config.recordVersion,  
    s&#34;[UnifiedLog partition=$topicPartition, dir=${dir.getParent}] &#34;,  
    None,  
    scheduler)  
  val producerStateManager = new ProducerStateManager(topicPartition, dir,  
    maxTransactionTimeoutMs, producerStateManagerConfig, time)  
  val isRemoteLogEnabled = UnifiedLog.isRemoteLogEnabled(remoteStorageSystemEnable, config, topicPartition.topic)  
  //diaoyong  
  val offsets = new LogLoader(  
    dir,  
    topicPartition,  
    config,  
    scheduler,  
    time,  
    logDirFailureChannel,  
    lastShutdownClean,  
    segments,  
    logStartOffset,  
    recoveryPoint,  
    leaderEpochCache.asJava,  
    producerStateManager,  
    numRemainingSegments,  
    isRemoteLogEnabled,  
  ).load()  
  val localLog = new LocalLog(dir, config, segments, offsets.recoveryPoint,  
    offsets.nextOffsetMetadata, scheduler, time, topicPartition, logDirFailureChannel)  
  new UnifiedLog(offsets.logStartOffset,  
    localLog,  
    brokerTopicStats,  
    producerIdExpirationCheckIntervalMs,  
    leaderEpochCache,  
    producerStateManager,  
    topicId,  
    keepPartitionMetadataFile,  
    remoteStorageSystemEnable,  
    logOffsetsListener)  
}
```


LogLoader的load()方法遍历当前分区dir，加载LogSegment，并存储到LogSegments中，流程如下：

```scala
private def loadSegmentFiles(): Unit = {  
  // load segments in ascending order because transactional data from one segment may depend on the  
  // segments that come before it  //1. 按升序加载  
  for (file &lt;- dir.listFiles.sortBy(_.getName) if file.isFile) {  
    //1.1 检查是否为index文件，若是则检查它的log文件是否存在  
    if (isIndexFile(file)) {  
      // if it is an index file, make sure it has a corresponding .log file  
      val offset = offsetFromFile(file)  
      val logFile = LogFileUtils.logFile(dir, offset)  
      if (!logFile.exists) {  
        warn(s&#34;Found an orphaned index file ${file.getAbsolutePath}, with no corresponding log file.&#34;)  
        Files.deleteIfExists(file.toPath)  
      }  
    } else if (isLogFile(file)) {  
      //1.2 加载log  
      // if it&#39;s a log file, load the corresponding log segment      val baseOffset = offsetFromFile(file)  
      val timeIndexFileNewlyCreated = !LogFileUtils.timeIndexFile(dir, baseOffset).exists()  
      //1.3 调用LogSegment.open()  
      val segment = LogSegment.open(  
        dir,  
        baseOffset,  
        config,  
        time,  
        true,  
        0,  
        false,  
        &#34;&#34;)  
      //1.4 进行完整性检查  
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
      //1.5 更新当前LogSegments  
      segments.add(segment)  
    }  
  }  
}
```

### LogSegment结构

LogSegment.open()是提供静态创建LogSegment的入口。

```java
public static LogSegment open(File dir, long baseOffset, LogConfig config, Time time, boolean fileAlreadyExists,  
                              int initFileSize, boolean preallocate, String fileSuffix) throws IOException {  
    int maxIndexSize = config.maxIndexSize;  
    return new LogSegment(  
        FileRecords.open(LogFileUtils.logFile(dir, baseOffset, fileSuffix), fileAlreadyExists, initFileSize, preallocate),  
        LazyIndex.forOffset(LogFileUtils.offsetIndexFile(dir, baseOffset, fileSuffix), baseOffset, maxIndexSize),  
        LazyIndex.forTime(LogFileUtils.timeIndexFile(dir, baseOffset, fileSuffix), baseOffset, maxIndexSize),  
        new TransactionIndex(baseOffset, LogFileUtils.transactionIndexFile(dir, baseOffset, fileSuffix)),  
        baseOffset,  
        config.indexInterval,  
        config.randomSegmentJitter(),  
        time);  
}
```


#### 成员对象

org.apache.kafka.storage.internals.log.LogSegment中存在以下属性：
```java

// 日志消息文件对象，FileRecords是文件关联对象
private final FileRecords log;  

//三个索引文件对象
private final LazyIndex&lt;OffsetIndex&gt; lazyOffsetIndex;  
private final LazyIndex&lt;TimeIndex&gt; lazyTimeIndex;  
private final TransactionIndex txnIndex;

//起始offset
private final long baseOffset;

//broker端参数log.index.interval.bytes值，用于控制LogSegment新增索引项的频率，默认写入4KB时新增一条索引项
private final int indexIntervalBytes;  

//LogSegment新增数据的扰动值，打散磁盘IO
private final long rollJitterMs; 

//写入日志的最新时间戳
private volatile OptionalLong rollingBasedTimestamp = OptionalLong.empty();
  
//最后一次更新索引至今，写入的字节数 
private int bytesSinceLastIndexEntry = 0;
```


##### FileRecords

log字段是FileRecords，该对象内部包括File对象、文件开始结束位置、文件大小、FileChannel，它的和新方法也较为简单：

```java
/**  
 * Append a set of records to the file. This method is not thread-safe and must be * protected with a lock. * * @param records The records to append  
 * @return the number of bytes written to the underlying file  
 */public int append(MemoryRecords records) throws IOException {  
    if (records.sizeInBytes() &gt; Integer.MAX_VALUE - size.get())  
        throw new IllegalArgumentException(&#34;Append of size &#34; &#43; records.sizeInBytes() &#43;  
                &#34; bytes is too large for segment with current file position at &#34; &#43; size.get());  
  
    int written = records.writeFullyTo(channel);  
    size.getAndAdd(written);  
    return written;  
}  
  
/**  
 * Commit all written data to the physical disk */public void flush() throws IOException {  
    channel.force(true);  
}  
  
/**  
 * Close this record set */public void close() throws IOException {  
    flush();  
    trim();  
    channel.close();  
}
```


在初始化LogSegment时，会调用FileRecords的open方法完成内部log字段的初始化：
```java
public static FileRecords open(File file,  
                               boolean mutable,  
                               boolean fileAlreadyExists,  
                               int initFileSize,  
                               boolean preallocate) throws IOException {  
    FileChannel channel = openChannel(file, mutable, fileAlreadyExists, initFileSize, preallocate);  
    int end = (!fileAlreadyExists &amp;&amp; preallocate) ? 0 : Integer.MAX_VALUE;  
    return new FileRecords(file, channel, 0, end, false);  
}
```


##### Index结构

```java
private final LazyIndex&lt;OffsetIndex&gt; lazyOffsetIndex;  
private final LazyIndex&lt;TimeIndex&gt; lazyTimeIndex;  
private final TransactionIndex txnIndex;
```


LogSegment有三个索引字段，其中offsetIndex和timeIndex使用LazyIndex包装类，以达到延迟初始化的效果，只有在第一次读取时，会将数据加载到内存中：

```java
public T get() throws IOException {  
    IndexWrapper wrapper = indexWrapper;  
    if (wrapper instanceof IndexValue&lt;?&gt;)  
        return ((IndexValue&lt;T&gt;) wrapper).index;  
    else {  
        lock.lock();  
        try {  
            if (indexWrapper instanceof IndexValue&lt;?&gt;)  
                return ((IndexValue&lt;T&gt;) indexWrapper).index;  
            else if (indexWrapper instanceof IndexFile) {  
                IndexFile indexFile = (IndexFile) indexWrapper;  
                IndexValue&lt;T&gt; indexValue = new IndexValue&lt;&gt;(loadIndex(indexFile.file));  
                indexWrapper = indexValue;  
                return indexValue.index;  
            } else  
                throw new IllegalStateException(&#34;Unexpected type for indexWrapper &#34; &#43; indexWrapper.getClass());  
        } finally {  
            lock.unlock();  
        }  
    }  
}
```


实际加载index文件使用mmap方式，将内核buffer映射到用户buffer，减少了内存拷贝的开销，这也是Kafka较为重要的优化点之一。

```java
private void createAndAssignMmap() throws IOException {  
    boolean newlyCreated = file.createNewFile();  
    RandomAccessFile raf;  
    if (writable)  
        raf = new RandomAccessFile(file, &#34;rw&#34;);  
    else  
        raf = new RandomAccessFile(file, &#34;r&#34;);  
  
    try {  
        /* pre-allocate the file if necessary */  
        if (newlyCreated) {  
            if (maxIndexSize &lt; entrySize())  
                throw new IllegalArgumentException(&#34;Invalid max index size: &#34; &#43; maxIndexSize);  
            raf.setLength(roundDownToExactMultiple(maxIndexSize, entrySize()));  
        }  
  
        long length = raf.length();  
        MappedByteBuffer mmap = createMappedBuffer(raf, newlyCreated, length, writable, entrySize());  
  
        this.length = length;  
        this.mmap = mmap;  
    } finally {  
        Utils.closeQuietly(raf, &#34;index &#34; &#43; file.getName());  
    }  
}
```

### Offset初始化

UnifiedLog在进行初始化时，同时也会更新内部的log start offset信息。

```scala
locally {
  //更新log start offset
  def updateLocalLogStartOffset(offset: Long): Unit = {  
    _localLogStartOffset = offset  
  
    if (highWatermark &lt; offset) {  
      updateHighWatermark(offset)  
    }  
  
    if (this.recoveryPoint &lt; offset) {  
      localLog.updateRecoveryPoint(offset)  
    }  
  }  
  
  initializePartitionMetadata()  
  updateLogStartOffset(logStartOffset)  
  updateLocalLogStartOffset(math.max(logStartOffset, localLog.segments.firstSegmentBaseOffset.orElse(0L)))  
  if (!remoteLogEnabled())  
    logStartOffset = localLogStartOffset()  
  maybeIncrementFirstUnstableOffset()  
  initializeTopicId()  
  
  logOffsetsListener.onHighWatermarkUpdated(highWatermarkMetadata.messageOffset)  
}
```

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-3-log-startup/  

