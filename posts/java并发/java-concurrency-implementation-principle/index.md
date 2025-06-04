# Java并发机制底层实现原理




## Volatile

### 定义

Java语言规范第3版中对volatile的定义如下：Java编程语言允许线程访问共享变量，为了能确保共享变量能被准确和一致的更新，线程应该确保通过排他锁单独获取这个变量。Java语言提供了volatile关键字，在某些情况下比锁要更加方便，如果一个变量被声明成volatile，Java线程内存模型确保所有线程看到的这个变量的值是一致的。



#### 实现原理



先看下面的CPU术语定义：

|    术语    |                             描述                             |
| :--------: | :----------------------------------------------------------: |
|  内存屏障  |        是一组处理器指令，用于实现对内存操作的顺序限制        |
|   缓冲行   | 缓存这两个可以分配的最小存储单位，处理器填写缓存线时会加载整个缓存线，需要使用多个主内存读周期 |
|  原子操作  |                  不可中断的一个或一系列操作                  |
| 缓存行填充 | 当处理器识别到内存中读取操作数是可缓存的，处理器读取整个缓存行到合适的缓存 |
|  缓存命中  | 如果进行高速缓存行填充操作的内存位置仍然是下次处理器访问的地址时，处理器从缓存中读取操作数，而不是从内存读取 |
|   写命中   | 当处理器将操作数写回到一个内存缓存中的区域中，它首先会检查这个缓存的内存地址是否在缓存行中，如果存在一个有效的缓存行，则处理器将这个操作数回写到缓存，而不是回写到内存，这个操作数被称为写命中 |
|   写缺失   |           一个有效的缓存行被写入到不存在的内存区域           |



```java
/**
 * @author ：L1nker4
 * @date ： 创建于  2020/4/7 20:34
 * @description： volatile测试
 */
public class Demo01 {

    private static volatile boolean stop = false;
    
    public static void main(String[] args) {
        stop = true;
        boolean b = stop;
    }
}
```



通过添加VM options打印程序汇编代码：

```
-XX:&#43;UnlockDiagnosticVMOptions -XX:&#43;LogCompilation -XX:&#43;PrintAssembly -Xcomp -XX:CompileCommand=dontinline,*Demo01.main -XX:CompileCommand=compileonly,*Demo01.main
```



如果提示以下内容，需要将`hedis-amd64.dll`放在`jre/bin/server`目录下。

```
Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
```



截取部分的汇编代码

```asm
Code:
[Disassembling for mach=&#39;i386:x86-64&#39;]
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} {0x000000001bcd2a70} &#39;main&#39; &#39;([Ljava/lang/String;)V&#39; in &#39;wang/l1n/volatile1/Demo01&#39;
  # parm0:    rdx:rdx   = &#39;[Ljava/lang/String;&#39;
  #           [sp&#43;0x40]  (sp of caller)
  0x0000000002a24760: mov    %eax,-0x6000(%rsp)
  0x0000000002a24767: push   %rbp
  0x0000000002a24768: sub    $0x30,%rsp
  0x0000000002a2476c: movabs $0x1bcd2be8,%rsi   ;   {metadata(method data for {method} {0x000000001bcd2a70} &#39;main&#39; &#39;([Ljava/lang/String;)V&#39; in &#39;wang/l1n/volatile1/Demo01&#39;)}
  0x0000000002a24776: mov    0xdc(%rsi),%edi
  0x0000000002a2477c: add    $0x8,%edi
  0x0000000002a2477f: mov    %edi,0xdc(%rsi)
  0x0000000002a24785: movabs $0x1bcd2a68,%rsi   ;   {metadata({method} {0x000000001bcd2a70} &#39;main&#39; &#39;([Ljava/lang/String;)V&#39; in &#39;wang/l1n/volatile1/Demo01&#39;)}
  0x0000000002a2478f: and    $0x0,%edi
  0x0000000002a24792: cmp    $0x0,%edi
  0x0000000002a24795: je     0x0000000002a247c3  ;*iconst_1
                                                ; - wang.l1n.volatile1.Demo01::main@0 (line 12)

  0x0000000002a2479b: movabs $0x76ba9ff38,%rsi  ;   {oop(a &#39;java/lang/Class&#39; = &#39;wang/l1n/volatile1/Demo01&#39;)}
  0x0000000002a247a5: mov    $0x1,%edi
  0x0000000002a247aa: mov    %dil,0x68(%rsi)
  0x0000000002a247ae: lock addl $0x0,(%rsp)     ;*putstatic stop
                                                ; - wang.l1n.volatile1.Demo01::main@1 (line 12)

  0x0000000002a247b3: movsbl 0x68(%rsi),%esi    ;*getstatic stop
                                                ; - wang.l1n.volatile1.Demo01::main@4 (line 13)

  0x0000000002a247b7: add    $0x30,%rsp
  0x0000000002a247bb: pop    %rbp
  0x0000000002a247bc: test   %eax,-0x25546c2(%rip)        # 0x00000000004d0100
                                                ;   {poll_return}
  0x0000000002a247c2: retq   
  0x0000000002a247c3: mov    %rsi,0x8(%rsp)
  0x0000000002a247c8: movq   $0xffffffffffffffff,(%rsp)
  0x0000000002a247d0: callq  0x0000000002a20860

```



可以看到在`mov %dil,0x68(%rsi)`写操作之后有`lock addl $0x0,(%rsp)`，lock前缀指令在处理器发生了两件事：

1. 将当前处理器缓存行的数据回写到系统内存。

数据写回内存是一个并发操作，如果另一个CPU也要写回内存，就会出现问题，所以需要锁，cache是486机器才引入的技术，所以在486以后P6处理器以前，是锁总线；在P6以后，如果访问的内存区域已经缓存在处理器内部，则不会声言Lock#信号，而是锁缓存&#43;缓存一致性协议（**cache coherency mechanism**）来保证指令的原子性。此操作称为**缓存锁定**。

2. 这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。

IA-32处理器和Intel 64处理器使用**缓存一致性协议（MESI）**维护内部缓存和其他处理器缓存的一致性。



&gt; Beginning with the P6 family processors, when the LOCK prefix is prefixed to an instruction and the memory area being accessed is cached internally in the processor, the LOCK# signal is generally not asserted. Instead, only the processor’s cache is locked. Here, the processor&#39;s cache coherency mechanism ensures that the operation is carried out atomically with regards to memory. 





## Synchronized



### 含义

synchronized实现同步的基础：Java每一个对象都可以作为锁，具体有以下三种表现：

1. 对于普通同步方法，锁是当前实例对象。
2. 对于静态同步方法，锁是当前类的Class对象。
3. 对于同步方法块，锁是括号里面的对象。

当一个线程试图访问同步代码块时，它首先必须得到锁，退出或者抛出异常时必须释放锁。



### 原理

```java
public class Demo02 {

    private static final Object object = new Object();

    public static void main(String[] args) {
        System.out.println(&#34;hello world&#34;);
    }
}
```



通过`javap -c Demo02.class`生成字节码指令

```
public class wang.l1n.volatile1.Demo02 {
  public wang.l1n.volatile1.Demo02();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object.&#34;&lt;init&gt;&#34;:()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String hello world
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return

  static {};
    Code:
       0: new           #5                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object.&#34;&lt;init&gt;&#34;:()V
       7: putstatic     #6                  // Field object:Ljava/lang/Object;
      10: return
}
```



将代码使用synchronized括起来之后生成的字节码指令如下：

```
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

可以看到字节码指令第5行-第15行被monitorenter和monitorexit包裹，执行到monitorenter会尝试获取对象的monitor，monitorexit会释放对象的monitor。



## 原子操作的实现原理

原子操作意为**不可被中断的一个或一系列操作**。



### 处理器层面实现

处理器提供总线锁定和缓存锁定两个机制来保证内存操作的原子性，总线锁定就是使用处理器的一个`LOCK #`信号，当一个处理器在总线上输出此信号时，其他处理器请求将被阻塞住。那么该处理器可以独占共享内存。

总线锁定开销较大，所以就有了缓存锁定。缓存锁定是指内存区域如果被缓存在处理器的缓存行中，并且在Lock操作期间被锁定，那么它执行锁操作回写到内存时，处理器不在总线上声言Lock信号，而实修改内部的内存地址，并允许它的缓存一致性协议来保证操作的原子性，缓存一致性协议会阻止同时修改由两个以上处理器缓存的内存区域数据。



### Java如何实现原子操作

Java使用锁和循环CAS的方式实现原子操作。



#### CAS

首先介绍一下CAS（Compare and Swap）操作，一个当前内存值V、旧的预期值A、即将更新的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则返回false。

 JVM中的CAS操作利用了提到的处理器提供的CMPXCHG指令实现的；循环CAS实现的基本思路就是循环进行CAS操作直到成功为止。



以`AtomicInteger`为例

```java
/**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
public final int incrementAndGet() {
      return unsafe.getAndAddInt(this, valueOffset, 1) &#43; 1;
}

//unsafe
public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 &#43; var4));

        return var5;
}

```

`getIntVolatile`通过偏移量获取到内存中变量值，`compareAndSwapInt`会比较获取的值与此时内存中的变量值是否相等，不相等则继续循环重复。整个过程利用CAS保证了对于value的修改的并发安全。



但是CAS存在以下问题：

1. ABA问题

CAS需要在操作值得时候检查是否发生变化，但是如果一个值是A，变成B，然后又变成A，CAS检查会发现没有变化。**AtomicStampedReference**来解决ABA问题:这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

2. 循环时间长开销大

自旋CAS如果长时间不成功，会给CPU带来较大得执行开销。

3. 只能保证一个共享变量的原子操作

对多个共享变量操作时，循环CAS就无法保证操作的原子性，可以使用锁来解决。

当然可以将多个共享变量合并成一个共享变量来操作，比如`i = 2;j = a`，合并为`ij = 2a`，然后CAS操作`ij`，从Java 1.5开始，JDK提供了`AtomicReference`类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。



### 锁机制

锁机制保证了只有获得锁的线程才能操作锁定的内存区域，JVM内部实现了很多锁机制，有偏向锁，轻量级锁和互斥锁，除了偏向锁，JVM实现锁的方式都用了循环CAS，当一个线程进入同步块的时候使用循环CAS的方式来获取锁，当它退出同步块的时候使用循环CAS释放锁。





## 参考

《Java并发编程的艺术》

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/java-concurrency-implementation-principle/  

