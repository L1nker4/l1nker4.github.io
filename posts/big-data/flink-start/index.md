# Flink入门



# 简介

Apache Flink是一个用于有状态的并行数据流的分布式计算系统，同时支持流式处理和批量处理。

实际数据分析应用都是面向无限数据流，三类通常使用有状态流处理实现的应用程序：
1. 事件驱动应用程序：使用特定的业务逻辑来提取事件流并处理事件。
	1. 实时推荐、行为模式检测、异常检测
2. 数据管道应用程序：将数据从A组件复制到B组件的ETL流程，pipeline包括多个源（source）和接收器（sink）。
	3. 实时数据仓库：数据实时清洗、归并、结构化
3. 数据流式分析应用程序：连续地提取事件流数据，并计算出最新结果并存储，用于查询。
	1. 用户行为等实时数据分析


常见的流式处理框架对比：

|       | Flink                                               | Spark Streaming              | Apache Storm    |
| ----- | --------------------------------------------------- | ---------------------------- | --------------- |
| 架构    | 主从模式                                                | 主从模式，依赖Spark，每个Batch处理都依赖主节点 | 主从模式，依赖ZK       |
| 处理方式  | Native                                              | Micro-Batch                  | Native          |
| 容错    | 基于Chandy-Lamport distributed snapshots checkpoint机制 | WAL与RDD机制                    | Record&#39;s ACK    |
| 处理模式  | 单条事件处理、时间窗口划分的所有事件                                  | 时间窗口内的所有时间                   | 单条事件处理          |
| 数据保证  | exactly once                                        | exactly once                 | at least once   |
| API支持 | high                                                | high                         | low             |
| 社区活跃度 | high                                                | high                         | medium          |
| 部署性   | 部署简单，仅依赖JRE                                         | 部署简单，仅依赖JRE                  | 依赖JRE和Zookeeper |




Flink优点：
1. 毫秒级延迟
2. 统一数据处理组件栈，可以处理不同类型的数据需求
3. 支持事件时间、接入时间、处理时间等概念
4. 基于轻量级分布式快照实现的容错
5. 支持有状态计算，灵活的state-backend（HDFS、内存、RocksDB）
6. 满足Exactly-once需求
7. 支持高度灵活的window操作
8. 带反压的连续流模型
9. 每秒千万级吞吐量
10. 易用性：提供了SQL、Table API、Stream API等方式



流式计算的时间窗口，涉及处理时间和事件时间，主要有以下区别：
- 处理时间提供了低延迟，但不能保证准确性
- 事件时间保证了结果准确性，实时性较差


结果保证：
- at-most-once：最多处理一次事件，事件可以被丢弃掉，也没有任何操作保证结果的正确性。
- at-least-once：允许事件被多次处理，以此保证结果的正确性
- exactly-once：恰好一次，最严格的保证，也最难实现，没有事件丢失，并且每个数据只处理一次。

Flink相关概念：
- Task：一个阶段中，多个功能相同的subTask集合，一个Task就是一个可以链接的最小算子链，这样可以减少线程间切换导致的开销，提高整体的吞吐量。
- SubTask：Flink任务最小执行单元（Java类），完成具体的计算逻辑。
- Slot：对计算资源进行隔离的单元，一个Slot中可以运行多个subTask
- State：运行过程中计算的中间结果
- Source：数据源
- Transformation：数据处理算子，包括map、filter、reduce
- Sink：Flink作业的数据存放点，例如MySQL、Kafka
- JobGraph：Flink运行任务的抽象图表达结构
	- 通过有向无环图DAG方式表达用户程序的执行流程
	- 不同接口程序的统一抽象表达：DataStream API、Flink SQL、Table API等
	- 转换流程：Application Code -&gt; StreamGraph -&gt; JobGraph


Task是概念上的任务，SubTask是实际提交给Task Slots的任务（单独的线程）。


部署模式：
- Session Mode（资源共享）：根据指定的资源初始化一个Flink集群，拥有固定数量的JobManager和TaskManager，所有Job在一个Runtime中运行。
	- 客户端通过RPC或者Rest API链接集群的管理节点
	- Deployer需要上传以来的Denpendences Jar
	- Deployer需要生成JobGraph，并提交到管理节点
	- JobManager的生命周期不受提交的Job影响。
- Per-Job Mode（deprecated）：基于资源协调框架为每一个提交的作业启动专属的Flink集群，作业完成后，资源将被关闭并清除，提供了良好的资源隔离能力。
	- `&gt; Per-job mode is only supported by YARN and has been deprecated in Flink 1.15. It will be dropped in [FLINK-26000](https://issues.apache.org/jira/browse/FLINK-26000). Please consider application mode to launch a dedicated cluster per-job on YARN.`
- Application Mode
	- 运行在Cluster上，而不在客户端
	- 每一个Application对应一个Runtime，Application中可以包含多个Job
	- 客户端无需将Denpendencies上传到JobManager，仅负责提交Job
	- main方法运行在JobManager中，将JobGraph的生成放在Cluster中运行

当前支持以下资源管理器部署集群：
- Standalone
- Hadoop Yarn
- Apache Mesos
- Docker
- Kubernetes



# 集群结构

Flink集群是类似于Master-Slave的结构，核心组件包括：
- JobManager：master节点，负责Flink作业的调度与执行，包括：Checkpoint Coordinator、jobGraph -&gt; Execution Graph、Task部署和调度、RPC通信(Actor System)、Job接收(Job Dispatch)、集群资源管理、TaskManager注册与管理
	- ResourceManager：负责Flink集群中的资源提供、回收与分配。ResourceManager是Task Slot的管理者，slot是Flink定义的处理资源单元。接收到JobManager的资源请求时，会将存在空闲slot的TaskManager分配给JobManager执行任务。
	- Dispatcher：提供了Restful接口用于提交Flink程序，并为每个提交的作业启动一个新的JobMaster。
	- JobMaster：负责单个JobGraph的执行。
- TaskManager：worker节点，负责实际SubTask的执行，并且缓存数据流。每个TaskManager都拥有Task Slot，slot由ResourceManager统一管理。功能包括：
	- Task Execution
	- Network Manager
	- Shuttle Environment manager
	- RPC system
	- Data Exchange
	- Offer Slots to JobManager
- Client：用户运行的main方法进程，包括以下功能：
	- JobGraph Generate
	- Execution Environment manager
	- Job Submit
	- RPC with JobManager
	- Cluster Deploy


# Flink编程模型

1. 创建Flink程序执行环境
2. 从数据源source读取一条或多条数据
3. 使用算子实现业务逻辑（transformation）
4. 将计算结果输出（sink）


# Flink API

Flink根据抽象程度，将API分为以下三类，分别适用不同的应用场景：
- SQL/ Table API
- DataStream API
- ProcessFuction


# DataStream API


## Source

#### 文件

从文件读取数据：

```java
DataStreamSource&lt;String&gt; source = env.readTextFile(&#34;/tmp/data&#34;);
```


### Socket

从Socket读取数据：

```java
DataStreamSource&lt;String&gt; source = env.socketTextStream(&#34;127.0.0.1&#34;, 8089);
```

可以通过nc等工具，本地调试验证。

### 本地变量

从变量读取数据：
```java
DataStreamSource&lt;String&gt; source = env.fromCollection(collection);


DataStream&lt;SensorReading&gt; source = env.fromElements(new Object())
```

### Kafka

从Kafka读取数据：

存在Kafka Source、Kafka Consumer两种读取方式。

```xml

&lt;flink.version&gt;1.13.2&lt;/flink.version&gt;


&lt;dependency&gt;  
    &lt;groupId&gt;org.apache.flink&lt;/groupId&gt;  
    &lt;artifactId&gt;flink-runtime-web_${scala.binary.version}&lt;/artifactId&gt;  
    &lt;version&gt;${flink.version}&lt;/version&gt;  
    &lt;scope&gt;provided&lt;/scope&gt;  
&lt;/dependency&gt;  
&lt;dependency&gt;  
    &lt;groupId&gt;org.apache.flink&lt;/groupId&gt;  
    &lt;artifactId&gt;flink-connector-kafka_2.12&lt;/artifactId&gt;  
    &lt;version&gt;${flink.version}&lt;/version&gt;  
&lt;/dependency&gt;  
  
&lt;dependency&gt;  
    &lt;groupId&gt;org.apache.flink&lt;/groupId&gt;  
    &lt;artifactId&gt;flink-connector-base&lt;/artifactId&gt;  
    &lt;version&gt;${flink.version}&lt;/version&gt;  
&lt;/dependency&gt;
```


```java
KafkaSource&lt;String&gt; kafkaSource = KafkaSource.&lt;String&gt;builder()  
        .setBootstrapServers(&#34;192.168.1.100:9092&#34;)  
        .setTopics(&#34;test_topic&#34;)  
        .setGroupId(&#34;flink-demo&#34;)  
        .setStartingOffsets(OffsetsInitializer.earliest())  
        .setValueOnlyDeserializer(new SimpleStringSchema())  
        .build();  
  
DataStreamSource&lt;String&gt; source = env.fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), &#34;Kafka Source&#34;);
```



```java
Properties consumerProperties = new Properties();  
consumerProperties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, &#34;192.168.1.100:9092&#34;);  
consumerProperties.put(ConsumerConfig.GROUP_ID_CONFIG, &#34;flink-demo&#34;);  
DataStreamSource&lt;String&gt; source =  
        env.addSource(new FlinkKafkaConsumer&lt;&gt;(&#34;test_topic&#34;, new SimpleStringSchema(), consumerProperties));
```


### 自定义Source

```java
import org.apache.flink.streaming.api.functions.source.RichParallelSourceFunction;

import java.util.Calendar;
import java.util.Random;

public class SensorSource extends RichParallelSourceFunction&lt;SensorReading&gt; {

    private boolean running = true;

    @Override
    public void run(SourceContext&lt;SensorReading&gt; srcCtx) throws Exception {

        Random rand = new Random();

        String[] sensorIds = new String[10];
        double[] curFTemp = new double[10];
        for (int i = 0; i &lt; 10; i&#43;&#43;) {
            sensorIds[i] = &#34;sensor_&#34; &#43; i;
            curFTemp[i] = 65 &#43; (rand.nextGaussian() * 20);
        }

        while (running) {
            long curTime = Calendar.getInstance().getTimeInMillis();
            for (int i = 0; i &lt; 10; i&#43;&#43;) {
                curFTemp[i] &#43;= rand.nextGaussian() * 0.5;
                srcCtx.collect(new SensorReading(sensorIds[i], curTime, curFTemp[i]));
            }

            Thread.sleep(100);
        }
    }

    @Override
    public void cancel() {
        this.running = false;
    }
}

```


```java
DataStream&lt;SensorReading&gt; sensorData = env.addSource(new SensorSource());
```


## Transformation

算子负责将一个或多个DataStream转换为新的DataStream,，转换算子可以分为以下四类：

- 基本转换算子：作用在数据流中的每一条单独数据上。
- KeyedStream转换算子，在数据有key的情况下，对数据应用转换算子。
- 多流转换算子：合并多条流为一条流，或者将一条流分割为多条流。
- 分布式转换算子：将重新组织流里面的事件。


### Map

Java Lambda Stream Map用法。

转换关系：将1个元素，转换为1个新元素。


Example：将输入元素自增1

```java
source.map(new MapFunction&lt;Integer, Integer&gt;() {
    @Override
    public Integer map(Integer value) throws Exception {
        return value &#43; 1;
    }
});
```


### FlatMap

转换关系：将1个元素，转换为任意个元素。

Example：将输入的String，按照` `进行拆分
```java
dataStream.flatMap(new FlatMapFunction&lt;String, String&gt;() {
    @Override
    public void flatMap(String value, Collector&lt;String&gt; out) throws Exception {
        for(String word: value.split(&#34; &#34;)){
            out.collect(word);
        }
    }
});
```


### Filter

按照设置的条件过滤，筛选出符合条件的元素。

Example: 过滤空串

```java
source.filter(new FilterFunction&lt;String&gt;() {  
	@Override  
	public boolean filter(String s) throws Exception {  
		return StringUtils.isNotBlank(s);  
	}
});
```


### KeyBy

将DataStream转换为KeyedStream：将key值相同的记录分配到相同的分区，类似于SQL的group by。

```java
source.keyBy(new KeySelector&lt;Tuple2&lt;String, Integer&gt;, String&gt;() {  
    @Override  
    public String getKey(Tuple2&lt;String, Integer&gt; value) throws Exception {  
        return value.f0;  
    }  
})
```


### Reduce

将KeyedStream转换为DataStream：将数据流中的元素与上一个Reduce后的元素进行合并产生一个新值，一般作用于有界数据流。

```java
keyedStream.reduce(new ReduceFunction&lt;Integer&gt;() {
    @Override
    public Integer reduce(Integer value1, Integer value2) throws Exception {
        return value1 &#43; value2;
    }
});
```


### Union

将两个以上的DataStream合并成一个。

```java
dataStream.union(stream1, stream2...);
```

### Connect

连接两个DataStream，允许两个流之间共享数据，与union有所区别：
1. union支持多个，connect仅支持两个
2. union要求流的数据类型一致，connect允许类型不一样。
3. connect允许两个流有不同的处理逻辑。


### CoMap

对被Connect后的ConnectedStream执行map

```java
connectedStreams.map(new CoMapFunction&lt;Integer, String, Boolean&gt;() {
    @Override
    public Boolean map1(Integer value) {
        return true;
    }

    @Override
    public Boolean map2(String value) {
        return false;
    }
});

```


### CoFlatMap

对被Connect后的ConnectedStream执行FlatMap

```java
connectedStreams.flatMap(new CoFlatMapFunction&lt;Integer, String, String&gt;() {
   @Override
   public void flatMap1(Integer value, Collector&lt;String&gt; out) {
       out.collect(value.toString());
   }

   @Override
   public void flatMap2(String value, Collector&lt;String&gt; out) {
       for (String word: value.split(&#34;,&#34;)) {
         out.collect(word);
       }
   }
});
```


## Partition

数据交换策略决定将输入流通过何种策略输出到下游算子的并行任务。

### Random

将数据随机分配到下游算子的并行任务中。

```java
source.shuffle();
```


### Rebalance

使用轮询方式将输入流平均分配到下游算子的并行任务中，它会与所有的下游算子联系，默认的交换策略。

```java
source.rebalance();
```


### Rescale

将数据发送到一部分的下游算子的并行任务中。

```java
source.rescale();
```


### Broadcast

将数据发送到下游算子的所有并行任务中。

```java
source.broadcast();
```


### Global

将数据发送到下游算子的第一个并行任务中。

```java
source.global();
```


### Custom

自定义分区策略，使用`partitionCustom()`


## Sink

Sink负责对计算结果的输出，Flink支持的Sink组件包括：Kafka、ES、Redis等。


### Print

将数据输出到控制台，用于调试阶段。

```java
dataStream.print();
```



### Kafka

```xml
&lt;dependency&gt;  
    &lt;groupId&gt;org.apache.flink&lt;/groupId&gt;  
    &lt;artifactId&gt;flink-connector-kafka_2.12&lt;/artifactId&gt;  
    &lt;version&gt;${flink.version}&lt;/version&gt;  
&lt;/dependency&gt;  
  
&lt;dependency&gt;  
    &lt;groupId&gt;org.apache.flink&lt;/groupId&gt;  
    &lt;artifactId&gt;flink-connector-base&lt;/artifactId&gt;  
    &lt;version&gt;${flink.version}&lt;/version&gt;  
&lt;/dependency&gt;
```

```java
// 定义 kafka sink
Properties produceProperties = new Properties();
produceProperties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, KAFKA_BROKER_SERVERS);
FlinkKafkaProducer&lt;String&gt; kafkaProducer = new FlinkKafkaProducer&lt;&gt;(
    &#34;test&#34;,
    new SimpleStringSchema(),
    produceProperties);

dataStream.addSink(kafkaProducer);
```



# Window

对于流式数据（无界），部分计算指标不存在实际意义，Window主要解决无界性问题，按照规则将其划分成有界数据，用于业务逻辑的处理。

Keyed vs Non-Keyed：定义Window时，需要确认数据流是否调用过keyBy()方法。

keyBy()后的Stream支持多个窗口并行计算，可以按照key拆分数据流并行计算，Non-Keyed Stream只支持并行度为1的计算。


模板：
```java

// Keyed Window
stream
       .keyBy(...)               &lt;-  仅 keyed 窗口需要
       .window(...)              &lt;-  必填项：&#34;assigner&#34;
      [.trigger(...)]            &lt;-  可选项：&#34;trigger&#34; (省略则使用默认 trigger)
      [.evictor(...)]            &lt;-  可选项：&#34;evictor&#34; (省略则不使用 evictor)
      [.allowedLateness(...)]    &lt;-  可选项：&#34;lateness&#34; (省略则为 0)
      [.sideOutputLateData(...)] &lt;-  可选项：&#34;output tag&#34; (省略则不对迟到数据使用 side output)
       .reduce/aggregate/apply()      &lt;-  必填项：&#34;function&#34;
      [.getSideOutput(...)]      &lt;-  可选项：&#34;output tag&#34;

// Non-Keyed Window
stream
       .windowAll(...)           &lt;-  不分组，将数据流中的所有元素分配到相应的窗口中
      [.trigger(...)]            &lt;-  指定触发器Trigger（可选）
      [.evictor(...)]            &lt;-  指定清除器Evictor(可选)
       .reduce/aggregate/process()      &lt;-  窗口处理函数Window Function
```


## Window schema

决定数据流入窗口的策略

### Tumbling Window

滚动窗口：固定大小、不可重叠

```java
// tumbling event-time windows  
input  
.keyBy(&lt;key selector&gt;)  
.window(TumblingEventTimeWindows.of(Time.seconds(5)))  
.&lt;windowed transformation&gt;(&lt;window function&gt;);
```


### Sliding Window

滑动窗口：固定大小、允许重叠

```java
// sliding event-time windows  
input  
.keyBy(&lt;key selector&gt;)  
.window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))  
.&lt;windowed transformation&gt;(&lt;window function&gt;);
```


### Session Window

当一个窗口在大于session gap的时间内没有接收到新数据，就会被关闭
i
会话窗口：不会重叠，也没有固定的起始时间，关闭取决于session gap的触发。



## Window Function

Stream指定窗口划分数据后，需要执行计算逻辑，通常包括以下三种。


### ReduceFunction

可以将输入流中前后两个元素进行合并操作，并输出类型相同的新元素。


### AggregateFunction

可以将输入元素进行增量聚合

AggregateFunction接口定义了以下4个方法：

```java
@PublicEvolving  
public interface AggregateFunction&lt;IN, ACC, OUT&gt; extends Function, Serializable {  
	//创建初始累加器
    ACC createAccumulator();  

	//累加输入元素到累加器
    ACC add(IN value, ACC accumulator);  

	//从累加器提取输出结果
    OUT getResult(ACC accumulator);  

	//合并累加器
    ACC merge(ACC a, ACC b);  
}
```


计算一个窗口内的平均数：

```java
public class AverageAccumulator {  
    long count;  
    long sum;  
}

public class WeightedAverage implements AggregateFunction&lt;Double, AverageAccumulator, Double&gt; {  
  
    public AverageAccumulator createAccumulator() {  
        return new AverageAccumulator();  
    }  
  
    public AverageAccumulator merge(AverageAccumulator a, AverageAccumulator b) {  
        a.count &#43;= b.count;  
        a.sum &#43;= b.sum;  
        return a;  
    }  
  
    public AverageAccumulator add(Double value, AverageAccumulator acc) {  
        acc.count &#43;= value;  
        acc.sum &#43;= value;  
        return acc;  
    }  
  
    public Double getResult(AverageAccumulator acc) {  
        return acc.sum / (double) acc.count;  
    }  
  
}

```



### ProcessWindowFunction

ProcessWindowFunction包含了以下内容：
- IN：输入元素类型
- OUT：输出元素类型
- KEY：用于分区字段的类型
- W：可被使用的窗口函数类型

使用起来更灵活，可以缓存窗口所有数据。


## Trigger

决定Window Function启动时机和窗口数据清理时机。


## Evictor

在Trigger触发后、调用Window Function之前或者之后从窗口中删除数据。

Flink内置三个Evictor：
- CountEvictor：仅记录用户指定数量的元素，一旦窗口元素超过指定值，多余元素从开头进行移除。
- DeltaEvictor：接收指定的threshold参数，计算窗口内最后一个元素和其他所有元素的差值，并移除差值&gt;= threshold的元素。
- TimeEvictor：接收interval参数，找到窗口中元素最大的timestamp，并移除比max - interval小的所有元素。


# Watermark

数据乱序场景时，当新数据输入进来触发窗口关闭后，旧数据找不到对应窗口就会丢失。

Watermark是用来度量数据event-time进展的字段，表明在watermark之前时间的数据都已经接收。


Watermark（水位线）：一条特殊的数据记录，以长整型值保存了一个时间戳，有两个基本属性：
1. 必须单调递增，以确保任务的时间再向前推进
2. 必须与数据的时间戳相关，后续的数据时间戳都必须大于它，可用于处理时间乱序数据流。


## Watermark策略

Flink需要从每个元素中提取可分配的时间戳，通常使用TimestampAssigner从元素某个字段提取出时间戳，同时通过指定WatermarkGenerator配置watermark生成策略。



Flink可以通过两种方式生成watermark：
1. 数据源source分配：通过SourceFunction分配watermark
2. 设置WatermarkStrategy


```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream&lt;MyEvent&gt; stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

DataStream&lt;MyEvent&gt; withTimestampsAndWatermarks = stream
        .filter( event -&gt; event.severity() == WARNING )
        .assignTimestampsAndWatermarks(&lt;watermark strategy&gt;);

withTimestampsAndWatermarks
        .keyBy( (event) -&gt; event.getGroup() )
        .window(TumblingEventTimeWindows.of(Time.seconds(10)))
        .reduce( (a, b) -&gt; a.add(b) )
        .addSink(...);
```



# 状态管理



## Sate Backend

State Backend是状态的管理组件，主要解决两个问题：
1. 管理本地状态，包括访问、存储和更新
2. checkpoint被激活时，决定状态同步的方式和位置



# 分区容错性

Flink作为一个分布式计算系统，需要有一套机制用于满足分区容错性，处理各类故障。

Flink主要提供了checkpoint与流重放相结合的机制来解决。

## checkpoint

Flink恢复机制的核心就是应用状态的一致检查点，checkpoint会每隔一段时间就保存一份snapshot，它包含了数据流中所有任务在某个时间点的状态信息。如果发生故障，使用最近的checkpoint恢复状态，并重启处理流程。

Flink基于Chandy-Lamport算法实现分布式快照的checkpoint的保存，该算法不会暂停整个应用程序，将保存流程和数据处理流程分离。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  
env.enableCheckpointing(5000);
```



TODO：算法细节


### Barrier

从checkpoint触发开始到checkpoint成功生成快照的过程中，如何判断哪些数据需要在当前checkpoint阶段进行保存，使用Barrier进行隔离。

特点：
- 随数据流传递的数据字段
- 以广播的方式向下游传递



### Storage Location

- JobManagerCheckpointStorage：状态保存在JobManager的内存堆
	- 通常用于本地开发测试
- FileSystemCheckpointStorage：状态保存到checkpointDirectory指向
	- 本地或远程


### 保留模式

作业取消后，checkpoint是否保留，提供了两个参数：
- DELETE_ON_CANCELLATION：Flink 作业取消后删除 checkpoint 文件
- RETAIN_ON_CANCELLATION：Flink 作业取消后保留 checkpoint 文件


```java
CheckpointConfig config = env.getCheckpointConfig();  
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```


### Recovery

作业出现故障后，Flink会停止数据流，将数据流重置到checkpoint最新生成的快照，包括两种恢复策略：
- Restart：决定了是否重启、何时重启作业
- Failover：决定应该重启哪些任务

&gt; https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/execution/task_failure_recovery/

## savepoint

允许应用程序从指定savepoint启动程序，创建savepoint的算法和checkpoint完全相同，可以认为是具有一些额外元数据的checkpoint，Flink不会自动创建，由外部程序触发创建操作。


## Backpressure

反压通常发生于的场景：短时间内系统接收数据的速率远高于处理数据的速率。因此需要对上游进行限速。

https://wuchong.me/blog/2016/04/26/flink-internals-how-to-handle-backpressure/
https://www.cnblogs.com/gentlescholar/p/15594153.html

---

> Author:   
> URL: http://localhost:1313/posts/big-data/flink-start/  

