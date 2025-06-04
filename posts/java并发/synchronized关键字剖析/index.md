# Synchronized关键字剖析


## 使用

在使用`Synchronized`关键字需要把握以下注意点：

- 一把锁只能同时被一个线程获取，没有获得锁的线程只能等待。
- 每一个实例都有自己的一个锁资源，存放于对象头中（2bit表示锁信息）



### 对象锁

- 同步代码块锁（可以指定锁定对象）
- 方法锁（默认锁定对象为this（当前实例对象））

```java
public void test() {
   synchronized (obj){
   System.out.println(&#34;hello&#34;);
   }
}

public synchronized void test() {
   System.out.println(&#34;hello&#34;);
}
```



### 类锁

`synchronized`修饰静态方法或指定锁对象为Class对象。

```java
public static synchronized void method() {
    //do something
}
synchronized(ObjectDemo.class){
    
}
```





## 理论基础

在操作系统进程管理中，对进程并发问题主要提供了两种解决方法：信号量和管程。在Java 1.5之前，提供的唯一并发原语就是管程，Java 1.5之后提供的JUC包也是以管程技术为基础的。



### 管程定义

&gt; 一个管程定义了一个数据结构和能为并发进程所执行的一组操作，这组操作能同步进程和改变管程中的数据。

通俗而言：管程（Monitor）是管理共享变量以及对共享变量的操作过程，让他们支持并发。在OS领域一般称为管程，Java中可以称为**监视器**（monitor）。



### MESA模型

MESA模型是当今广泛使用的MESA模型，Java管程的实现参考的也是MESA模型。并对其进行了精简。Java内置的管程只有一个条件变量。

如下图所示：管程X将共享变量queue、入队操作于出队操作封装起来。如果线程A和线程B访问共享变量queue，只能通过调用管程提供的`enq()`和`deq()`来实现。两个方法保证互斥性，，只允许一个线程进入管程并操作。该模型能实现并发编程中的互斥问题。

![管程](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/java/concurrent/sync/MESA.jpg)

下图为MESA管程模型示意图，框中即是封装的管程，所有线程通过入口等待队列进入管程。管程还引入了条件变量的概念，**每一个条件变量都对应一个等待队列**。管程的同步主要通过`Condition`（条件变量）实现。`Condition`可以执行`wait()`和`signal()`。

假设线程T1执行出队操作，同时有个前提条件：队列不为空，这是条件变量。如果T1进入管程发现队列为空，则会在条件变量的等待队列进行等待。调用`wait()`实现。此刻允许其它线程进入管程。

此时线程T2执行入队操作，入队成功后，队列不空条件对于T1已经满足，T2调用`notify()`来通知T1。通知他条件已满足。

![MESA管程模型](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/java/concurrent/sync/MESA1.jpg)



### 两个操作



#### wait

MESA模型提供了一个特有的编程范式，通过循环检查条件调用`wait()`。管程模型中：条件满足后，如何通知相关线程。管程要求同一时刻只能有一个线程能执行，那么上述问题中T1，T2谁执行呢？

在MESA中，T2通过`notify()`通知完后，继续执行，T1从条件变量的等待队列进入入口等待队列中。

```java
while(条件不满足) {
	wait();
}
```



#### signal

尽量使用`notifyAll()`，如果满足以下三个条件则可以使用`notify()`：

- 所有等待线程拥有相同的等待条件
- 所有等待线程被唤醒后，执行相同的操作
- 只需要唤醒一个线程





## 实现



### JVM字节码层面

从JVM层面来看，主要通过两个字节码指令实现，`monitorenter`与`monitorexit`。这两个字节码需要指定一个对象引用作为参数。这个对象引用就是monitor object。它就是synchronized传入的对象实例，该对象充当着维护了mutex以及顶层父类`Object`提供的`wait/notify`机制。

```java
public class wang.l1n.volatile1.Demo02 {
  public wang.l1n.volatile1.Demo02();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object.&#34;&lt;init&gt;&#34;:()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #2                  // Field object:Ljava/lang/Object;
       3: dup
       4: astore_1
       5: monitorenter
       6: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: ldc           #4                  // String hello world
      11: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      14: aload_1
      15: monitorexit
      16: goto          24
      19: astore_2
      20: aload_1
      21: monitorexit
      22: aload_2
      23: athrow
      24: return
    Exception table:
       from    to  target type
           6    16    19   any
          19    22    19   any

  static {};
    Code:
       0: new           #6                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object.&#34;&lt;init&gt;&#34;:()V
       7: putstatic     #2                  // Field object:Ljava/lang/Object;
      10: return
}
```







### JVM实现层面



每个Java对象都关联一个Monitor对象，如果使用`synchronized`给对象上锁，该对象的`MarkWord`中就被设置指向Monitor对象的指针。

Java对象在堆内存中存储，其中`Mark Word`中`2bit`存储了锁标识，Java的顶层父类`Object`定义了`wait()`，`notify()`，`notifyAll()`方法，这些方法的具体实现，依赖于`ObjectMonitor`模式，这是JVM内部基于C&#43;&#43;实现的一套机制，基本原理如下图所示：

![ObjectMonitor](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/java/concurrent/sync/object%20monitor.png)

- Entry Set：从未获得过Monitor得线程，排队竞争Monitor
- owner：同一时刻，只有一个线程持有Monitor
- Wait Set：曾经持有Monitor的线程，通过`Object.wait()`主动进入wait set

当一个线程需要获取Object锁时，会判断当前owner是否被持有，未持有则此线程会成为锁的owner，否则进入`wait Set`中进行等待。

该线程可以通过调用wait方法将锁释放，进入`wait Set`中进行等待，其他线程此刻可以获得锁，从而使得之前不成立的条件变量（Condition）成立，这样之前阻塞的线程就可以重新进入`Entry Set`取竞争锁。



- 持有Monitor的线程t1检到条件变量c不符合，则执行`wait()`，使自己：
  - 主动释放Monitor资源
  - 进入Wait Set，挂起自己
- 线程t2发现t1符合条件变量c之后，执行`notify()`，使得：
  - s继续正常执行，直到执行完毕释放Monitor或者主动`wait()`
  - s释放Monitor后，Wait Set中的t1重新竞争获取Monitor



HotSpot VM通过[ObjectMonitor](https://hg.openjdk.java.net/jdk/jdk/file/896e80158d35/src/hotspot/share/runtime/objectMonitor.hpp)实现该机制，该类包含以下关键字段：

- _EntryList：对应 entry set
- _WaitSet：对应 wait set
- _owner：指向持有 Monitor 线程的指针
- _recursions：重入次数，获得同一个Monitor 加1，释放同一个Monitor减1，为0，说明释放了Monitor。
- _count：近似等于 _EntryList &#43; _WaitSet



### 互斥锁存储结构

`MarkWord`结构如下：

```
|--------------------------------------------------------------------|--------------------|
| Mark Word (64 bits) 							| State |
|--------------------------------------------------------------------|--------------------|
| unused:25 | hashcode:31 | unused:1 | age:4 | biased_lock:0 	| 01 | Normal |
|--------------------------------------------------------------------|--------------------|
| thread:54 | epoch:2 | unused:1 | age:4 | biased_lock:1 		| 01 | Biased |
|--------------------------------------------------------------------|--------------------|
| ptr_to_lock_record:62 							| 00 | Lightweight Locked |
|--------------------------------------------------------------------|--------------------|
| ptr_to_heavyweight_monitor:62 | 					10 | Heavyweight Locked |
|--------------------------------------------------------------------|--------------------|
|							 | 11 			| Marked for GC |
|--------------------------------------------------------------------|--------------------|

```









## JVM中锁的优化

在JDK 1.6之前，synchronized的实现会调用`Object`的enter和exit，这种锁被称为重量级锁，需要从用户态切换到内核态执行，十分消耗性能，在JDK1.6之后，对锁的实现引入了大量的优化，比如锁粗化（Lock Coarsening），锁消除（Lock Elimination），轻量级锁（Lightweight Locking），偏向锁（Biased Locking），适应性自旋（Adaptive Spinning）等优化技术来减少锁的性能开销。



JDK 1.6中的Synchronized同步锁，一共有四种状态：无锁，偏向锁，轻量级锁，重量级锁，数据存储在`Mark Word`中。

它会随着竞争情况逐渐升级，但是不可以降级，目的是为了提供获取锁和释放锁的效率。

### 无锁

无锁没有对资源进行锁定，所有线程都能访问并修改同一个资源，但同时只有一个线程能修改成功。

无锁的特点是修改操作在循环内进行，线程会不断尝试修改共享资源。如果没有冲突就修改成功并退出，否则就会继续循环尝试。CAS原理就是无锁的实现。



### 偏向锁

偏向锁是指一段同步代码一直被一个线程所访问（**重入锁机制**），那么该线程就会自动获得锁，降低获得锁的代价。

当一个线程通过同步代码块获得锁的时候，会在`Mark Word`中存储锁偏向的线程ID。在线程进入或退出同步代码块时不再通过CAS操作来加锁解锁，而是检查`Mark Word`中是否存储着指向当前线程的偏向锁。

偏向锁只有遇到其他线程竞争偏向锁时，持有偏向锁的线程才会释放偏向锁，线程不会主动释放偏向锁。

偏向锁在JDK 6以后是默认启用的，可以通过`-XX:UseBiasedLocking=false`关闭，关闭之后，程序默认进入轻量级锁状态。



撤销偏向锁的时机：

- 调用对象的hashCode
- 其他线程使用对象锁
- 调用wait/notify



- **批量重偏向**：对象被多个线程访问，但是未造成竞争，当对象偏向某线程后，**在规定时间内，若另一个线程也尝试获取资源，偏向锁升级为轻量级锁，并且这个次数达到阈值20次时，这个对象就不会升级为轻量级锁，而直接改变偏向线程。**
- **批量重撤销**：当上述次数达到阈值40次，JVM认为这个对象继续使用偏向锁会影响性能，取消偏向锁机制。





偏向锁的几个参数：

```
-XX:BiasedLockingBulkRebiasThreshold = 20   // 默认偏向锁批量重偏向阈值
-XX:BiasedLockingBulkRevokeThreshold = 40   // 默认偏向锁批量撤销阈值
-XX:BiasedLockingDecayTime					//重偏向的阈值事件
-XX:&#43;UseBiasedLocking // 使用偏向锁，jdk6之后默认开启
-XX:BiasedLockingStartupDelay = 0 // 延迟偏向时间, 默认不为0，jvm启动多少ms以后开启偏向锁机制（此处设为0，不延迟）
```





### 轻量级锁

使用场景：多线程访问时间错开（没有竞争），使用轻量级锁。

轻量级锁是指当锁是偏向锁的时候，被另一个线程访问，偏向锁就会升级为轻量级锁，其他线程通过自旋的方式尝试获取锁，不会阻塞。从而提高性能。

轻量级锁并不是替代重量级锁的，而是对在大多数情况下同步块并不会有竞争出现提出的一种优化。它可以减少重量级锁对线程的阻塞带来地线程开销。



### 重量级锁

若当前只有一个等待线程，则该线程通过自旋进行等待，但是当自旋超过一定次数，或是一个线程在持有锁，一个在自旋，又有第三个线程访问时，轻量级锁升级为重量级锁。



|    锁    |                             优点                             |                             缺点                             |              使用场景              |
| :------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :--------------------------------: |
|  偏向锁  | 加锁和解锁不需要CAS操作，没有额外的性能消耗，和执行非同步方法相比仅存在纳秒级的差距 |        如果线程间存在锁竞争，会带来额外的锁撤销的消耗        | 适用于只有一个线程访问同步快的场景 |
| 轻量级锁 |              竞争的线程不会阻塞，提高了响应速度              |     如线程始终得不到锁竞争的线程，使用自旋会消耗CPU性能      | 追求响应时间，同步快执行速度非常快 |
| 重量级锁 |               线程竞争不使用自旋，不会消耗CPU                | 线程阻塞，响应时间缓慢，在多线程下，频繁的获取释放锁，会带来巨大的性能消耗 |   追求吞吐量，同步快执行速度较长   |





### 自旋锁与自适应自旋锁

​		在Java中，自旋锁是指尝试获取锁的线程不会立即阻塞，而实采用循环的方式去获取锁，这样做的好处是减少线程上下文切换的消耗。

但是自旋锁本身是有缺点的，它不能代替阻塞，自旋虽然避免了上下文切换的开销，但它要占用处理器时间，如果锁被占用的时间很短，自旋等待的效果很好，但是如果锁占用时间国产，自旋只会白白浪费处理器资源。所以自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，通过**-XX:PreBlockSpin**修改）没有成功获得锁，就挂起线程，停止自旋。

自旋锁的实现原理是CAS算法。自旋锁在JDK 1.4.2引入，使用**-XX:UseSpinning**开启，JDK 6开始默认开启，并且引入了自适应的自旋锁。

自适应意味着自旋的时间不再固定，而实由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么JVM会认为这次自选也是很有可能再次成功，进而它将自旋等待持续更长的时间。如果某个锁自旋很少成功获得，那么就会直接省略掉自旋过程，直接阻塞线程。

在自旋锁中，有三种常见的锁形式：TicketLock、CLHlock、MCSlock



### 锁消除

​		锁销除指的是虚拟机即时编译器在运行时，对一些代码上要求同步，但是对被检测到不可能存在共享数据竞争的锁进行消除。锁销除的主要判定依据是来源于逃逸分析的数据支持。（JVM会判断一段程序中的同步明显不会逃逸出去从而被其他线程访问到，那么JVM把它们当成线程独有的数据。）

例如下述代码，在JDK1.5之后，Javac编译器会对该段代转换成`StringBuilder`对象的`append`操作进行字符串连接。`StringBuilder`非线程安全，但是JVM判断该段代码不会逃逸，所以会进行锁销除操作。

```java
public static String demo(String s1, String s2) {
    String s = s1 &#43; s2;
    return s;
}
```



### 锁粗化

当连续的一系列操作会对一个对象反复加锁解锁，会消耗大量CPU资源，JVM会检测到这种情况，并将加锁粗化到整个方法。例如下述代码。

```java
public static String demo(String s1, String s2) {
    StringBuilder sb = new StringBuilder();
    sb.append(s1);
    sb.append(s2);
    return sb.toString();
}
```



### Synchronized与Lock

Lock是JUC的顶层接口，用户能通过其实现互斥同步功能。Lock在实现上并未使用到synchronized，而是利用了volatile的可见性。

Lock与synchronized相比，提供了更加方便的API。ReentrantLock是Lock的最常用的实现类，提供了以下功能：

- 等待可中断：持有锁的线程长时间不释放锁，等待的线程可以选择放弃等待。
- 公平锁：根据申请锁的顺序依次获取锁，会使得性能下降。synchronized为非公平锁，ReentrantLock默认为非公平锁，但是可以指定为公平锁。
- 锁可以绑定多个Condition。





---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/synchronized%E5%85%B3%E9%94%AE%E5%AD%97%E5%89%96%E6%9E%90/  

