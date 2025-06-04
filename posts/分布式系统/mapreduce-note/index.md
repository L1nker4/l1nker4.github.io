# MapReduce论文阅读笔记




## 存在的问题

Google所面临的问题：大数据处理任务庞大，如何通过分布式系统完成并行计算、分发数据、处理错误？

为了解决这个问题，需要设计一个新的抽象模型，用来表述我们需要执行的计算。不用关心底层的实现细节，包括并行计算、容错、数据分布、负载均衡等方面。





## 编程模型

MapReduce编程模型的原理：利用一个输入的**key/value pair**集合来产生一个输出的**key/value pair**集合。

自定义的Map函数接受一个**key/value pair**输入，然后产生一个中间**key/value pair**集合。会将相同key和对应多个value值集合在一起传递给reduce函数。

自定义的Reduce函数接受上面的集合，合并这些value值，形成一个新的value值集合。



下面是一个计算大文档集合中每个单词出现的次数的案例：

```
map(String key, String value):
	//key : document name
	//value: document contents
	foreach word w in value:
		emitintermediate(w, &#34;1&#34;);
		
reduce(String key, Iterator values):
	//key : a word
	//values: a list of counts
	int result = 0;
	foreach v in values:
		result &#43;= ParseInt(v);
	emit(AsString(result));
```

Map计函数计算文档中`(word,count)`这样的**key/value pair**，Reduce函数把Map函数产生的计数累加起来。

Map Reduce函数可以抽象成以下形式：

```
map(k1, v1) -&gt; list(k2, v2)
reduce(k2, list(v2)) -&gt; list(v2)
```





### 其他案例

- 分布式的grep：Map输出匹配某个模式的一行，Reduce将中间数据处理再输出。
- 计算URL访问频率：Map处理日志，输出`(URL, 1)`，Reduce将中间数据累加处理，产生`(URL, totalCount)`再输出。



## 实现

将输入数据自动分割成M块数据片段，Map函数在多台机器上并行处理。Reduce调用也是在多台机器上并行处理。MapReduce实现的全部流程如图所示：

![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/distribute%20system/MapReduce/execution-overview.jpg)





### master数据结构

Master存储每个Map和Reduce任务的状态，以及Worker机器的标识。

Master像一个数据管道，存储已完成的Map任务的计算结果。并将这些信息推送给Reduce任务。





### 容错



- worker故障
  - master会对worker进行心跳检测，如果在指定时间内没有收到返回信息，会将worker标记为失效。
  - 所有由这个失效worker完成的Map任务需要重新分配给其他的worker。
  - 当这个Map任务被调度到worker B执行时，会通知执行Reduce任务的worker从这台机器上读取数据。
- master故障
  - 让master定期的将上述的数据结构写入磁盘，如果master出现故障，可以让新的master从这个**checkpoint**恢复。
  - 由于只有一个master进程，如果master失效，需要中止MapReduce运算。



### 存储位置



MapReduce的master在调度Map任务时会考虑输入文件的位置信息，尽量将一个Map任务调度在有数据拷贝的机器上执行。这样可以减少网络带宽的消耗。



### 任务粒度

Map拆分成 M个片段，Reduce拆分成R个片段执行。

但是实际上对M和R的取值有一定的客观限制：

- master必须执行$O(M &#43; R)$次调度
- 需要在内存中保存$O(M * R)$个状态

R由用户指定的，实际使用时需要选择合适的M，以使得每一个独立任务都处理大约16M~64M的输入数据。R值应该设置为worker机器数量的倍数。





### 备用任务



如果一个worker花了很长时间才完成最后几个Map或Reduce任务，会导致MapReduce操作总的执行时间超过预期。产生以上现象的原因有很多。

因此需要使用一个通用的机制来减少这种现象，当一个MapReduce操作接近完成的时候，master调度备用任务进程来处理剩下的任务。



## 改良的功能



### 分区函数

选择什么函数来进行对数据进行分区？

`hash(key) mod R`能产生非常平衡的分区，但是，其它分区函数对一些key的分区效果较好，例如输入的key值是URL，我们希望每个主机的所有条目保持在同一个输出文件中。



### 顺序保证

需要保证Reduce的生成顺序。





### Combiner函数

某些情况下，Map函数产生的中间key值的重复数据会占很大的比重，因此，允许用户指定一个可选的combiner函数，该函数首先在本地将重复数据进行一次合并，然后再通过网络发送出去。





### 输入和输出的类型

可以预定义输入/输出的数据类型。



### 跳过损坏的记录



有时，用户程序中的Bug会导致Map/Reduce函数在处理记录的时候发生crash，常用的做法是修复Bug后再执行MapReduce操作。

但是很多时候，忽略一些有问题的记录是可以接受的，因此提供了一种执行模式，MapReduce会检测哪些记录导致了crash，并且会跳过这些记录不做处理。

每个worker进程都设置了信号处理函数来捕获内存段错误和总线错误。在执行MapReduce之前，通过全局变量保存记录序号。如果用户程序触发了一个系统信号，消息处理函数会通过UDP向master发送处理的最后一条记录的序号，当master看到处理某条记录失败多次后，master标记此条记录将不会被处理。



### 本地运行

需要本地进行debug来测试。



### 状态信息

master可以使用HTTP服务器来显示状态信息，用户可以监控各类信息。包括任务完成数量、任务处理数量、执行时间、资源占用等。





### 计数器

MapReduce提供计数功能，需要用户在程序中创建一个计数器对象，在Map/Reduce函数中相应的增加计数器的值。

计数器机制对于MapReduce操作的完整性检查十分有用。某些时候，用户需要确保输出的**key/value pair**精确等于输入的**key/value pair**



---

> Author:   
> URL: http://localhost:1313/posts/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/mapreduce-note/  

