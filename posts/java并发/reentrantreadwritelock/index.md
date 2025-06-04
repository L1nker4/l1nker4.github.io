# ReentrantReadWriteLock源码分析


# 简介

ReentrantReadWriteLock分为读锁和写锁两个实例，读锁是共享锁，可被多个读线程同时使用，写锁是独占锁。持有写锁的线程可以继续获取读锁，反之不行。

Doug Lea 将持有写锁的线程，去获取读锁，之后释放读锁，最后释放写锁，从写锁降级为读锁的过程称为**锁降级（Lock downgrading）**。

但是，**锁升级**是不可以的。线程持有读锁的话，在没释放的情况下不能去获取写锁，因为会发生**死锁**。



# 类的继承关系

ReentrantReadWriteLock实现了ReadWriteLock接口，该接口定义了两个方法，分别返回读锁和写锁。

```java
public class ReentrantReadWriteLock implements ReadWriteLock, java.io.Serializable {}
    
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
```



# 类成员属性

```java
//读锁
private final ReentrantReadWriteLock.ReadLock readerLock;

//写锁
private final ReentrantReadWriteLock.WriteLock writerLock;

//Sync是AQS的实现类
final Sync sync;

//Unsafe实例
private static final sun.misc.Unsafe UNSAFE;

//获取Thread.tid的内存偏移值
private static final long TID_OFFSET;

static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class&lt;?&gt; tk = Thread.class;
            TID_OFFSET = UNSAFE.objectFieldOffset
                (tk.getDeclaredField(&#34;tid&#34;));
        } catch (Exception e) {
            throw new Error(e);
        }
}
```



# 构造方法

默认的构造方法创建非公平策略的`ReentrantReadWriteLock`，传入`true`则可以创建公平策略的`ReentrantReadWriteLock`。

```java
public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```



# 内部类

`ReentrantReadWriteLock`共有五个内部类，其基本结构如下：

- Sync：AQS的实现类
  - FairSync：公平策略
  - NofairSync：非公平策略
- ReadLock：读锁
- WriteLock：写锁



## Sync

该类继承自`AQS`抽象类，`ReentrantReadWriteLock`的大部分操作都交给`Sync`对象进行处理。



### HoldCounter

该类配合读锁使用，属性定义如下：

- count： 代表某个读线程重入的次数
- tid：获取当前线程的TID属性

```java
static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
}
```



### ThreadLocalHoldCounter

该类继承自`ThreadLocal`，并重写了`initialValue()`，`ThreadLocal`可以将线程与对象相关联，`get`得到的值都是`initialValue()`生成的`HoldCounter`对象。

```java
static final class ThreadLocalHoldCounter
            extends ThreadLocal&lt;HoldCounter&gt; {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
}
```



### 类的成员属性



```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 版本序列号
    private static final long serialVersionUID = 6317671515068378041L;        
    // 高16位为读锁，低16位为写锁
    static final int SHARED_SHIFT   = 16;
    // 读锁单位
    static final int SHARED_UNIT    = (1 &lt;&lt; SHARED_SHIFT);
    // 读锁最大数量
    static final int MAX_COUNT      = (1 &lt;&lt; SHARED_SHIFT) - 1;
    // 写锁最大数量
    static final int EXCLUSIVE_MASK = (1 &lt;&lt; SHARED_SHIFT) - 1;
    // 本地线程计数器
    private transient ThreadLocalHoldCounter readHolds;
    // 缓存的计数器
    private transient HoldCounter cachedHoldCounter;
    // 第一个读线程
    private transient Thread firstReader = null;
    // 第一个读线程的计数
    private transient int firstReaderHoldCount;
}
```



### 构造方法

```java
Sync() {
    		//本地线程计数器
            readHolds = new ThreadLocalHoldCounter();
    		//设置AQS的state
            setState(getState()); // ensures visibility of readHolds
}
```



### sharedCount

该方法将`c`无符号右移16位，得到的值为读锁的线程数量，因为`c`的高16位代表读锁，低16位代表写锁数量。

也可以通过方法命名看出来，读锁是共享模式，写锁是独占模式。

```java
static int sharedCount(int c)    { return c &gt;&gt;&gt; SHARED_SHIFT; }
```



### exclusiveCount

该方法表示返回占有写锁的线程数量，通过`state`与`(1 &lt;&lt; 16) - 1`进行与运算，其等价于`state % 2 ^ 16`，因为写锁数量由`state`的低16位表示。

```java
static int exclusiveCount(int c) { return c &amp; EXCLUSIVE_MASK; }
```



### tryRelease

此方法用于释放写锁，通过这个调用链，可以清楚的看出`AQS`在并发类中的重要性，这也体现出了`AQS`的设计精髓，通过模板模式，将具体操作延迟到子类去实现。

```java
//WriteLock
public void unlock() {
            sync.release(1);
}

//AQS中定义
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null &amp;&amp; h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
}

protected final boolean tryRelease(int releases) {
    		//判断当前线程是否是独占线程
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
    		//nextc为释放资源后的写锁资源数量
            int nextc = getState() - releases;
    		//判断释放后的写锁数量是否为0
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                //为0说明当前没有线程独占
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
}
```



### tryReleaseShared

`ReadLock`释放锁的流程，与`WriteLock`释放类似。

```java
//ReadLock
public void unlock() {
            sync.releaseShared(1);
}

protected final boolean tryReleaseShared(int unused) {
    		//获取当前线程
            Thread current = Thread.currentThread();
    		//如果当前线程是第一个读线程
            if (firstReader == current) {
                // assert firstReaderHoldCount &gt; 0;
                //判断读线程占用资源数是否为1
                if (firstReaderHoldCount == 1)
                    //为1则将第一个读线程置为空
                    firstReader = null;
                else
                    //不然就--
                    firstReaderHoldCount--;
            } else {
                //到这段说明：当前线程不是第一个读线程
                HoldCounter rh = cachedHoldCounter;
                //如果计数器为空，或者计数器中tid存储的不是当前线程
                if (rh == null || rh.tid != getThreadId(current))
                    //将计数器设置为当前线程计数器
                    rh = readHolds.get();
                //获取count
                int count = rh.count;
                if (count &lt;= 1) {
                    //count &lt;= 1 则将ThreadLocal中的值删除
                    readHolds.remove();
                    if (count &lt;= 0)
                        throw unmatchedUnlockException();
                }
                //更新计数器
                --rh.count;
            }
    		//CAS自旋进行更新state
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```



### tryAcquire

此方法用于写线程获取写锁，基本流程如下：

- 首先会获取state，判断是否为0
  - 若为0，表示此时没有读锁线程。再判断写线程是否应该被阻塞，而在非公平策略下总是不会被阻塞，在公平策略下会进行判断(判断同步队列中是否有等待时间更长的线程，若存在，则需要被阻塞，否则，无需阻塞)，之后在设置状态state，然后返回true。
  - 若不为0，则表示此时存在读锁或写锁线程，若写锁线程数量为0或者当前线程不是独占锁线程，则返回false，表示不成功，否则，判断写锁线程的申请资源数量 &#43; 现有的写线程数量是否大于了`MAX_COUNT`，若是，则抛出`Error`，否则，设置状态`state`，返回true，表示成功。

```java
//WriteLock
public void lock() {
            sync.acquire(1);
}

protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
    		//c为state
            int c = getState();
    		//w为写锁的线程数量
            int w = exclusiveCount(c);
    		//c == 0代表当前没有读锁线程或写锁线程
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //写线程数量为0或者当前线程没有占有独占资源
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //如果申请的数量 &#43; 现有的写线程 &gt; MAX_COUNT
                if (w &#43; exclusiveCount(acquires) &gt; MAX_COUNT)
                    throw new Error(&#34;Maximum lock count exceeded&#34;);
                // Reentrant acquire
                setState(c &#43; acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c &#43; acquires))
                return false;
    		//设置独占线程
            setExclusiveOwnerThread(current);
            return true;
        }
```



### tryAcquireShared

此方法被读线程用于获取读锁，基本流程如下：

- 首先判断写锁是否为0并且当前线程不占有独占锁，直接返回。
- 否则，判断读线程是否需要被阻塞并且读锁数量是否小于最大值并且比较设置状态成功
  - 若当前没有读锁，则设置第一个读线程firstReader和firstReaderHoldCount
  - 若当前线程线程为第一个读线程，则增加firstReaderHoldCount
  - 否则，将设置当前线程对应的HoldCounter对象的值。
- 如果下列三个条件不满足(读线程是否应该被阻塞、小于最大值、比较设置成功)则会执行`fullTryAcquireShared`。

```java
protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
    		//如果当前有写线程，并且独占线程不是当前线程
            if (exclusiveCount(c) != 0 &amp;&amp;
                getExclusiveOwnerThread() != current)
                return -1;
    		//读锁数量
            int r = sharedCount(c);
    		//如果当前线程需要被阻塞，并且读锁数量小于MAX_COUNT,并且CAS设置state成功
            if (!readerShouldBlock() &amp;&amp;
                r &lt; MAX_COUNT &amp;&amp;
                compareAndSetState(c, c &#43; SHARED_UNIT)) {
                //读锁为0
                if (r == 0) {
                    //设置第一个读线程和计数器
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    //当前线程为第一个读线程
                    //计数器加1
                    firstReaderHoldCount&#43;&#43;;
                } else {
                    //设置当前线程对应的HoldCounter对象的值
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count&#43;&#43;;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```





## NofairSync

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```



## FairSync

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```



## ReadLock



### 基本信息

`ReadLock`拥有一个内部变量`sync`，构造方法用于初始化`sync`，可以联系`ReentrantReadWriteLock`的构造方法一起看。

```java
public static class ReadLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -5992448646407690164L;
        private final Sync sync;

        /**
         * Constructor for use by subclasses
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
}
```



### 核心方法

从核心方法`lock()`和`unlock()`中可以看出其具体实现都是交给`sync`进行实现。读操作由于是共享的，所以它使用的是`AQS`的共享模式实现的。

```java
public void lock() {
            sync.acquireShared(1);
}

//相应中断的lock
public void lockInterruptibly() throws InterruptedException {
            sync.acquireSharedInterruptibly(1);
}

//尝试获取lock
public boolean tryLock() {
            return sync.tryReadLock();
}

public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

public void unlock() {
            sync.releaseShared(1);
}
```



## WriteLock

`WriteLock`和`ReadLock`类似，不同的是，写操作是独占的，因此它使用`AQS`的独占模式实现。

```java
public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;

        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        public void lock() {
            sync.acquire(1);
        }

        public void lockInterruptibly() throws InterruptedException {
            sync.acquireInterruptibly(1);
        }

        public boolean tryLock( ) {
            return sync.tryWriteLock();
        }

        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireNanos(1, unit.toNanos(timeout));
        }

        public void unlock() {
            sync.release(1);
        }
```



# Reference

- https://juejin.cn/post/6844903663488483336

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/reentrantreadwritelock/  

