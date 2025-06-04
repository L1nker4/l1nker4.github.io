# JVM之内存分配策略（五）


## 内存分配与回收策略

Java技术体系的自动内存管理，最根本性的目标是自动化解决两个问题：自动给对象分配内存，以及自动回收分配给对象的内存。

对象的内存分配，就是在堆上分配（也有可能经过JIT编译后被拆散为标量空间类型间接地在栈上分配），新生对象主要分配在新生代中，少数情况（大小超过阈值）会被分配在老年代，分配规则不固定，取决于当前使用的垃圾收集器与JVM参数设置。

### 对象优先在Eden分配
大多数情况下，对象在新生代Eden去中分配，当Eden区没有足够的空间进行分配时，虚拟机将发起一次Minor GC。

* Minor GC：回收新生代（包括 Eden 和 Survivor 区域），因为 Java 对象大多都具备朝生夕灭的特性，所以 Minor GC 非常频繁，一般回收速度也比较快。
* Major GC / Full GC: 回收老年代，出现了 Major GC，经常会伴随至少一次的 Minor GC，但这并非绝对。Major GC 的速度一般会比 Minor GC 慢 10 倍 以上。 


### 大对象直接进入老年代
大对象就是需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串，或者元素数量很庞大的数组。

JVM提供了一个`-XX:PretenureSizeThreshold`，指定大于该设置值的对象直接在老年代分配，这样做的目的是避免在Eden区和两个Survivor区之间来回复制，产生大量内存复制操作。


### 长期存活的对象进入老年代

为了决策哪些存活对象存储在新生代，哪些对象存储在老年代，，JVM给每个对象定义了一个对象年龄计数器，存储在对象头中，对象通常在Eden区诞生，如果经历一次Miror GC后仍然存活，并且能被Survivor容纳，该对象就会被移动到Survivor，并将该对象年龄设为1岁，对象每经历一次Miror GC，年龄就增加1岁，当年龄达到一定程度（默认15），就会被移动到老年代，对象年龄阈值可以通过`-XX:MaxTenuringThreshold`设置。

### 动态对象年龄判定
如果当前新生代的Survivor中，相同年龄的所有对象大小的综合大于Survivor空间的一半，年龄大于或等于该年龄的对象可以进入老年代，无需等到MaxTenuringThreshold 中要求的年龄。

### 空间分配担保
JDK 6 Update 24 之前，在发生Miror GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那这一次的Miror GC可以确保是安全的，如果不成立，则虚拟机会查看 HandlePromotionFailure 值是否设置为允许担保失败， 如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小， 如果大于，将尝试进行一次 Minor GC,尽管这次 Minor GC 是有风险的； 如果小于，或者 HandlePromotionFailure 设置不允许冒险，那此时也要改为进行一次 Full GC。

JDK 6 Update 24 之后的规则变为：  
 只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小，就会进行 Minor GC，否则将进行 Full GC。


### 哪些情况会使JVM进行Full GC？
- System.gc() 方法的调用

此方法是建议JVM进行Full GC，我们可以通过 -XX:&#43; DisableExplicitGC 来禁止调用 System.gc()。

- 老年代空间不足

老年代空间不足会触发 Full GC操作，若进行该操作后空间依然不足，则会抛出如下错误：
` java.lang.OutOfMemoryError: Java heap space `

- 永久代空间不足
JVM 规范中运行时数据区域中的方法区，在 HotSpot 虚拟机中也称为永久代（Permanet Generation），存放一些类信息、常量、静态变量等数据，当系统要加载的类、反射的类和调用的方法较多时，永久代可能会被占满，会触发 Full GC。如果经过 Full GC 仍然回收不了，那么 JVM 会抛出如下错误信息：&lt;br&gt;
`java.lang.OutOfMemoryError: PermGen space `

- CMS GC 时出现 promotion failed 和 concurrent mode failure
promotion failed，就是上文所说的担保失败，而 concurrent mode failure 是在执行 CMS GC 的过程中同时有对象要放入老年代，而此时老年代空间不足造成的。

- 统计得到的Minor GC晋升到旧生代的平均大小大于老年代的剩余空间

---

> Author:   
> URL: http://localhost:1313/posts/jvm/jvm05-memory-allocate/  

