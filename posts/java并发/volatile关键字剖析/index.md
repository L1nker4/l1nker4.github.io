# Volatile关键字剖析




## 定义

&gt; The Java programming language allows threads to access shared variables (§17.1).As a rule, to ensure that shared variables are consistently and reliably updated, a thread should ensure that it has exclusive use of such variables by obtaining a lock that, conventionally, enforces mutual exclusion for those shared variables.The Java programming language provides a second mechanism, volatile fields,that is more convenient than locking for some purposes.A field may be declared volatile, in which case the Java Memory Model ensures that all threads see a consistent value for the variable (§17.4).

上述定义摘自《The Java Language Specification Java SE 8 Edition》，从语言规范中给出的定义可以总结出：

- 变量被`volatile`修饰后，JMM能确保所有的线程看到的这个变量的值是一致的。（可见性）



## 作用



### 防止重排序

在单例模式中，并发环境通常使用`Double Check Lock`来解决问题，案例如下：

```java
public class Singleton {
    private static Singleton singleton;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (null == singleton) {
            synchronized (Singleton.class) {
                if (null == singleton) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```



上述案例在`singleton = new Singleton();`这句代码，可以分解为三个步骤：

1. 分配内存空间
2. 初始化对象
3. 将内存空间的地址赋值给引用

但是，由于编译器或者处理器为了提高程序性能，会对指令进行重排序，会将上述三个步骤按以下步骤执行：

1. 分配内存空间
2. 将内存空间的地址赋值给引用
3. 初始化对象

如果按照这个流程，多线程环境下。如果线程A通过单例模式获取实例，此时获取的对象并未完成初始化，线程B访问该实例，会存在空指针异常。

通过对`singleton`字段添加`volatile`关键字可以解决这个问题。



### 实现可见性

可见性问题主要指一个线程修改了共享变量值，另一个线程看不到修改过的值，该问题主要原因是JMM要求每一个线程都拥有自己的工作内存（相当于高速缓存）。`volatile`关键字可以使得线程禁用该高速缓存。案例如下：变量b作为条件变量，b为`true`则进入死循环，在线程执行`start()`后，将b改为false并不能结束死循环，因为线程1一直从高速缓存中读取该值。通过添加`volatile`可以解决这个问题。

```java
public class VolatileDemo03 {

    boolean b = true;

    void m() {
        System.out.println(&#34;start&#34;);
        while (b) {

        }
        System.out.println(&#34;end&#34;);
    }

    public static void main(String[] args) {
        final VolatileDemo03 t = new VolatileDemo03();
        new Thread(new Runnable() {
            @Override
            public void run() {
                t.m();
            }
        }).start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t.b = false;
    }
}
```



### 无法保证原子性

#### i&#43;&#43;问题

`i&#43;&#43;`在指令层面主要分为三步：

- load i
- increment
- store i

volatile无法保证三个操作具有原子性。

假设线程1将i的值`load`，存入工作内存中，再放到寄存器A中，然后执行`increment`自增1，此时线程2执行同样的操作，然后`store`回写主内存，此时线程1的工作内存的i缓存失效，重新从主内存中读取该值，读到11，接着线程1执行`store`将寄存器A的值11回写主内存，这样就出现了线程安全问题。

```java
public class VolatileDemo04 {

    volatile int i;

    public void addI() {
        i&#43;&#43;;
    }

    public static void main(String[] args) throws InterruptedException {
        final VolatileDemo04 test = new VolatileDemo04();
        for (int i = 0; i &lt; 1000; i&#43;&#43;) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test.addI();
                }
            });
            thread.start();
        }
        Thread.sleep(10000);
        System.out.println(test.i);
    }
}
```



#### long,double变量

32位JVM将`long`和`double`变量的操作分为高32位和低32位两部分，普通的long/double的r/w操作可能不是原子性的，使用volatile可以保证单次r/w操作是原子性的。

&gt; For the purposes of the Java programming language memory model, a single write to a non-volatile `long` or `double` value is treated as two separate writes: one to each 32-bit half. This can result in a situation where a thread sees the first 32 bits of a 64-bit value from one write, and the second 32 bits from another write.
&gt;
&gt; Writes and reads of volatile `long` and `double` values are always atomic.
&gt;
&gt; Writes to and reads of references are always atomic, regardless of whether they are implemented as 32-bit or 64-bit values.
&gt;
&gt; Some implementations may find it convenient to divide a single write action on a 64-bit `long` or `double` value into two write actions on adjacent 32-bit values. For efficiency&#39;s sake, this behavior is implementation-specific; an implementation of the Java Virtual Machine is free to perform writes to `long` and `double` values atomically or in two parts.
&gt;
&gt; Implementations of the Java Virtual Machine are encouraged to avoid splitting 64-bit values where possible. Programmers are encouraged to declare shared 64-bit values as `volatile` or synchronize their programs correctly to avoid possible complications.



------



## 实现



### 可见性



**volatile的可见性是基于内存屏障（Memory Barrier）实现的。**

- 内存屏障：一组CPU指令，用于实现对内存操作的顺序限制。

  - 对volatile变量写指令后添加写屏障
  - 对volatile变量读指令前添加读屏障

  

前置知识：

- 缓存行（Cache Line）：CPU Cache的最小单位，通常大小为64字节（取决于CPU），缓存行可能会导致伪共享问题。

```java
public class Demo01 {

    private static volatile boolean stop = false;
    
    public static void main(String[] args) {
        stop = true;
    }
}
```

通过添加VM options打印程序汇编代码：

```
-XX:&#43;UnlockDiagnosticVMOptions -XX:&#43;LogCompilation -XX:&#43;PrintAssembly -Xcomp -XX:CompileCommand=dontinline,*VolatileDemo05.main -XX:CompileCommand=compileonly,*VolatileDemo05.main
```



如果提示以下内容，需要将`hedis-amd64.dll`放在`jre/bin/server`目录下。

```
Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
```



volatile变量操作部分汇编代码如下：

```java
  0x0000000003941c9b: movabs $0x76baa20b0,%rsi  ;   {oop(a &#39;java/lang/Class&#39; = &#39;wang/l1n/concurrent/volatiledemo/VolatileDemo05&#39;)}
  0x0000000003941ca5: mov    $0x1,%edi
  0x0000000003941caa: mov    %dil,0x68(%rsi)
  0x0000000003941cae: lock addl $0x0,(%rsp)     ;*putstatic stop
```



可以看到在`mov %dil,0x68(%rsi)`写操作之后有`lock addl $0x0,(%rsp)`，lock前缀指令在处理器发生了两件事：

1. 将当前处理器缓存行的数据回写到系统内存。
2. 写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。

为了提高处理速度，CPU不直接与内存进行通信，而是先将数据缓存到Cache（L1、L2）中，回写内存的时机由系统决定。

那么如果对volatile变量进行写操作，则不会经过Cache，而是直接将缓存行回写到主内存。

数据写回内存是一个并发操作，如果另一个CPU也要写回内存，就会出现问题，所以需要锁定。cache是486机器才引入的技术，所以在486以后P6处理器以前，是锁总线；在P6以后，如果访问的内存区域已经缓存在处理器内部，则不会声言Lock#信号，而是锁缓存&#43;缓存一致性协议（**cache coherency mechanism**）来保证指令的原子性。此操作称为**缓存锁定**。

IA-32处理器和Intel 64处理器使用**缓存一致性协议（MESI）**维护内部缓存和其他处理器缓存的一致性。

MESI的核心思想：CPU写数据时，如果发现变量是共享变量（其它CPU也存在该变量的副本），会通知其它CPU将该变量的缓存行设置为无效状态，因此其它CPU读该变量时，会从内存重新读取。

&gt; Beginning with the P6 family processors, when the LOCK prefix is prefixed to an instruction and the memory area being accessed is cached internally in the processor, the LOCK# signal is generally not asserted. Instead, only the processor’s cache is locked. Here, the processor&#39;s cache coherency mechanism ensures that the operation is carried out atomically with regards to memory. 



volatile和MESI的区别：

- 缓存一致性：硬件层面的问题，指的是多核CPU中的多个Cache缓存的一致性问题，MESI解决的是这个问题。
- 内存一致性：多线程程序中访问内存中变量的一致性，volatile解决这个问题。

通俗来说，**没有一点关系**。

------



### 有序性



#### volatile的happens-before规则

`happens-before`规则中有一条关于volatile变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。



#### volatile禁止重排序

JMM使用内存屏障来解决，下表是JMM针对编译器制定的重排序规则表：

| 能否重排序 |           |            |            |
| :--------: | :-------: | :--------: | :--------: |
| 第一个操作 | 普通读/写 | volatile读 | volatile写 |
| 普通读/写  |           |            |     NO     |
| volatile读 |    NO     |     NO     |     NO     |
| volatile写 |           |     NO     |     NO     |

为了能实现该表中制定的顺序，编译器再生成字节码文件时，会插入内存屏障来禁止重排序。

------

| 内存屏障指令 |                            说明                             |
| :----------: | :---------------------------------------------------------: |
|  StoreStore  |        禁止上面的普通写和下面的 volatile 写重排序。         |
|  StoreLoad   | 防止上面的 volatile 写与下面可能有的 volatile 读/写重排序。 |
|   LoadLoad   |    禁止下面所有的普通读操作和上面的 volatile 读重排序。     |
|  LoadStore   |    禁止下面所有的普通写操作和上面的 volatile 读重排序。     |



## 参考

《Java并发编程的艺术》

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/volatile%E5%85%B3%E9%94%AE%E5%AD%97%E5%89%96%E6%9E%90/  

