# LockSupport源码分析




## 简介

&gt; Basic thread blocking primitives for creating locks and other synchronization classes.   -- Java Doc

`LockSupport`是用来创建锁和其它同步类的基本线程阻塞原语。底层依赖`Unsafe`实现，我们可以在其它的并发同步工具类的实现中看到该类的使用。`LockSupport`提供了`park()`和`unpark()`方法来分别实现阻塞线程和唤醒线程，每个使用`LockSupport`的线程都有一个`permit`，该值默认为0，取值0,1。

- `unpark()`：如果`permit`当前值为0，将其自增1。
- `park()`：如果当前`permit`为1，将其自减1并立即返回，如果为0，直接阻塞。

这两个方法不会有`Thread.suspend` 和`Thread.resume`所可能引发的死锁问题，因为`permit`存在，调用 `park `的线程和另一个试图将其 `unpark `的线程之间的竞争将保持活性。

如果调用线程被中断，那么`park`将会返回。`park`方法可能在任何时间**no reason**地返回，因此通常在重新检查返回条件地循环里调用此方法。在某种意义上，`park`是**busy wait**（忙则等待）的一种优化，减少了自旋对性能的消耗。当时必须与`unpark`配合使用才会更加高效。

`park`还提供了支持`blocker`参数的方法，`blocker`对象在线程受阻塞时被记录，用于允许监视和诊断工具确定线程被阻塞的原因。提供了`getBlocker(Thread t)`来访问`blocker`。



下面是`Java Docs`中的示例用法：一个先进先出非重入锁类的基本框架：

```java
class FIFOMutex {
    private final AtomicBoolean locked = new AtomicBoolean(false);
    private final Queue&lt;Thread&gt; waiters
      = new ConcurrentLinkedQueue&lt;Thread&gt;();
 
    public void lock() {
      boolean wasInterrupted = false;
      Thread current = Thread.currentThread();
      waiters.add(current);
 
      // Block while not first in queue or cannot acquire lock
      while (waiters.peek() != current ||
             !locked.compareAndSet(false, true)) {
        LockSupport.park(this);
        if (Thread.interrupted()) // ignore interrupts while waiting
          wasInterrupted = true;
      }

      waiters.remove();
      if (wasInterrupted)          // reassert interrupt status on exit
        current.interrupt();
    }
 
    public void unlock() {
      locked.set(false);
      LockSupport.unpark(waiters.peek());
    }
  }}
```



------



## 源码解读



### 成员变量



- UNSAFE：用于进行内存级别操作的工具类。
- parkBlockerOffset：存储`Thread.parkBlocker`的内存偏移地址，记录线程被谁阻塞的。用于线程监控和分析工具用来定位原因的。可以通过`getBlocker`获取到阻塞的对象。
- SEED：存储`Thread.threadLocalRandomSeed`的内存偏移地址
- PROBE：存储`Thread.threadLocalRandomProbe`的内存偏移地址
- SECONDARY：存储`Thread.threadLocalRandomSecondarySeed`的内存偏移地址

```java
private static final sun.misc.Unsafe UNSAFE;
private static final long parkBlockerOffset;
private static final long SEED;
private static final long PROBE;
private static final long SECONDARY;
```



### 构造方法

不允许实例化，只能通过调用静态方法来完成操作。

```java
private LockSupport() {} // Cannot be instantiated.
```



### 静态代码块

通过反射机制获取`Thread`类的`parkBlocker`字段，然后通过`UNSAFE.objectFieldOffset`获取到`parkBlocker`在内存的偏移量。

&gt; Q：为什么不通过get/set方式获取某个字段？
&gt;
&gt; A：parkBlocker在线程处于阻塞状态下才会被赋值，此时直接调用线程内的方法，线程不会作出回应的。

```java
static {
        try {
            //获取unsafe实例
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class&lt;?&gt; tk = Thread.class;
            parkBlockerOffset = UNSAFE.objectFieldOffset
                (tk.getDeclaredField(&#34;parkBlocker&#34;));
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField(&#34;threadLocalRandomSeed&#34;));
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField(&#34;threadLocalRandomProbe&#34;));
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField(&#34;threadLocalRandomSecondarySeed&#34;));
        } catch (Exception ex) { throw new Error(ex); }
    }
```



### setBlocker

对给定的线程`t`的`parkBlocker`赋值。

```java
private static void setBlocker(Thread t, Object arg) {
        // Even though volatile, hotspot doesn&#39;t need a write barrier here.
        UNSAFE.putObject(t, parkBlockerOffset, arg);
    }
```



### getBlocker

返回线程`t`的`parkBlocker`对象。

```java
public static Object getBlocker(Thread t) {
        if (t == null)
            throw new NullPointerException();
        return UNSAFE.getObjectVolatile(t, parkBlockerOffset);
    }
```



### park



park方法阻塞线程，发生以下情况时，当前线程会继续执行：

- 其它线程调用`unpark`方法唤醒该线程
- 其它线程中断当前线程
- **no reason**地返回

该方法有两个重载版本。

&gt; Q：为什么调用两次setBlocker方法？

A：调用`park`方法时，当前线程首先设置好`parkBlocker`字段，然后调用`UNSAFE.park`方法，此时，当前线程阻塞，第二个`setBlocker`无法执行，过了一段时间，该线程的`unpark`方法被调用，该线程拿到`permit`后执行，将该线程的`blocker`字段置空。



```java
public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }

public static void park() {
        UNSAFE.park(false, 0L);
    }
```



### parkNanos

阻塞当前线程，最长不超过`nanos`纳秒

```java
public static void parkNanos(long nanos) {
        if (nanos &gt; 0)
            UNSAFE.park(false, nanos);
    }

public static void parkNanos(Object blocker, long nanos) {
        if (nanos &gt; 0) {
            Thread t = Thread.currentThread();
            setBlocker(t, blocker);
            UNSAFE.park(false, nanos);
            setBlocker(t, null);
        }
    }
```



### parkUntil

该方法表示在限定的时间内将阻塞。

```java
public static void parkUntil(long deadline) {
        UNSAFE.park(true, deadline);
    }
```



### unpark

如果给定线程的`permit`不可用，则将其置为可用，如果该线程阻塞，则将它解除阻塞状态。否则，保证下一次调用`park`不会受阻塞。如果给定线程尚未启动，则无法保证该操作有任何效果。

```java
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```





## 参考

[Java SE Doc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/LockSupport.html#park--)



---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/locksupport%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/  

