# JVM内存问题排查流程


## 确认问题现象

可以通过服务状态，监控面板、日志信息、监控工具（VisualVM）等，确认问题类型：



1. 内存使用率居高不下、内存缓慢增加、OOM等
2. 频繁GC：Full GC等



发现问题不建议重启，留存状态。



## 保留数据



### heapdump文件

```
#arthas导出方式
heapdump /tmp/dump.hprof

#jmap命令保存整个Java堆
jmap -dump:format=b,file=heap.bin &lt;pid&gt; 

#jmap命令只保存Java堆中的存活对象, 包含live选项，会在堆转储前执行一次Full GC
jmap -dump:live,format=b,file=heap.bin &lt;pid&gt;

#jcmd命令保存整个Java堆,Jdk1.7后有效
jcmd &lt;pid&gt; GC.heap_dump filename=heap.bin

#在出现OutOfMemoryError的时候JVM自动生成（推荐）节点剩余内存不足heapdump会生成失败
-XX:&#43;HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.bin

#编程的方式生成
使用HotSpotDiagnosticMXBean.dumpHeap()方法

#在出现Full GC前后JVM自动生成，本地快速调试可用
-XX:&#43;HeapDumpBeforeFullGC或 -XX:&#43;HeapDumpAfterFullGC
```

JVM参数：
```
-XX:printGCDetails -XX:&#43;UseConcMarkSweepGC -XX:&#43;HeapDumpOnOutOfMemoryError 
```



### GC日志

JVM启动参数如下：
```
# Java8及以下
-XX:&#43;PrintGCDetails -XX:&#43;PrintGCDateStamps -Xloggc:&lt;path&gt;

# Java9及以上
-Xlog:gc*:&lt;path&gt;:time
```

可以通过EasyGC进行日志分析。



### 服务日志

通常由日志组件（Promtail &amp; loki）进行采集、存储、展示。



## 实际问题


### 堆内存溢出

问题现象：



1. OutOfMemoryError


### 直接内存溢出

问题现象：top指令中相关Java进程占用的RES超过了-Xmx的大小，并且内存占用不断上升。

问题定位方法：



1. 可通过开启NMT（**-XX:NativeMemoryTracking=detail**）定位问题
2. jcmd查看直接内存情况：jcmd pid VM.native_memory detail



NIO、Netty等常用直接内存，Netty可以通过`-Dio.netty.leakDetectionLevel`开启



### 栈空间溢出



问题现象：



1. StackOverflow
2. OutOfMemoryError：unable to create new native thread



问题定位方法：



1. 分析Java调用栈
2. 开启Linux coredump日志

### 频繁GC



问题现象：通过监控面板、jstat等方式，发生频繁GC现象：



1. Minor GC：回收新生代区域
2. Major GC：回收老年代区域
3. Full GC：回收整个堆区和方法区
4. Mixed GC：回收整个新生代和部分老年代，部分GC实现（G1）




## 工具使用

### jstat
用于查看虚拟机垃圾回收的情况，主要是堆内存使用、垃圾回收次数和占用时间
```shell
jstat -gcutil  -h 20 pid 1000 100

各列分别为：S0使用率、S1使用率、Eden使用率、Old使用率
Method方法区使用率、CCS压缩使用率
YGC年轻代垃圾回收次数、YGCT年轻代垃圾回收占用时间
FGC全局垃圾回收次数、FGCT全局垃圾回收消耗时间
GCT总共垃圾回收时间
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/406326/1675906799781-537c7346-3eb0-4b60-91e9-64ca73e0b215.png#averageHue=%230b0806&amp;clientId=u200445ac-f1ae-4&amp;from=paste&amp;height=241&amp;id=ub7219716&amp;originHeight=316&amp;originWidth=960&amp;originalType=binary&amp;ratio=1&amp;rotation=0&amp;showTitle=false&amp;size=63366&amp;status=done&amp;style=none&amp;taskId=ue414be00-c4c6-4321-ba74-055bf4a9997&amp;title=&amp;width=731)


### jmap
用于保存虚拟机内存镜像到文件中，可以使用JVisualVM或MAT进行分析

```shell
jmap -dump:format=b,file=filename.hprof pid
```

也可以通过运行参数，当发生OOM时，主动保存堆dump文件。
```shell
-XX:&#43;HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof
```

### JMC
Java Mission Control：用于追踪热点代码和热点线程


## 附JVM参数

#### -Xms512m
- 意义： 设置堆内存初始值大小。
- 默认值：如果未设置，初始值将是老年代和年轻代分配制内存之和。
#### -Xmx1024m

- 意义： 设置堆内存最大值。

#### -XX:NewRatio=n

- 意义：设置老年代和年轻代的比例。比如：-XX:NewRatio=2 

-XX:SurvivorRatio
年轻代分配：默认情况下Eden、From、To的比例是8:1:1。SurvivorRatio默认值是8，如果SurvivorRatio修改成4，那么其比例就是4:1:1

#### -XX:MaxRAMPercentage=85.0

堆的最大值百分比


#### -Xalwaysclassgc

在全局垃圾回收期间始终执行动态类卸载检查。


####  -Xaggressive

Enables performance optimizations and new platform exploitation that are expected to be the default in future releases.

#### -XX:&#43;PrintGCDetails
打印GC日志

#### -XX:&#43;PrintGCTimeStamps
打印GC时间戳

#### -Xloggc

GC日志存储位置



#### -verbose:class
用于同时跟踪类的加载和卸载 

#### -XX:&#43;TraceClassLoading
单独跟踪类的加载

#### -XX:&#43;TraceClassUnloading 
单独跟踪类的卸载

#### -XX:NativeMemoryTracking=\[off | summary | detail \]
开启直接内存追踪


#### -XX:MaxDirectMemorySize
最大直接内存

#### -XX:MaxJavaStackTraceDepth
栈帧输出数量


---

> Author:   
> URL: http://localhost:1313/posts/jvm/jvm%E5%86%85%E5%AD%98%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5%E6%B5%81%E7%A8%8B/  

