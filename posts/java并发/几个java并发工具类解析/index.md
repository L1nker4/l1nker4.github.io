# 几个Java并发工具类解析


# CountDownLatch



## 简介

&gt; A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

只有当N个线程执行完毕，并且进行`countDown`操作时，才允许`await`的线程继续执行。否则该线程挂起。

适用情况：一个线程需要等待其他N个线程执行完毕，再继续执行，是join的替代。

## 构造方法

参数`count`为计数值，传入`AQS`的实现类`Sync`设置成AQS的`state`。

```java
public CountDownLatch(int count) {
        if (count &lt; 0) throw new IllegalArgumentException(&#34;count &lt; 0&#34;);
        this.sync = new Sync(count);
}
```



## Sync

通过继承`AQS`从而完成同步的核心功能。

```java
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
		
    	//构造方法
        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```



## 核心方法

- **countDown**：将count值减1
- **await**：调用**await**的线程会被挂起，直到`count`为0才继续执行，允许中断

```java
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
}

public void countDown() {
        sync.releaseShared(1);
}

public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```



## 使用案例

```java
class Driver { // ...
   void main() throws InterruptedException {
     CountDownLatch startSignal = new CountDownLatch(1);
     CountDownLatch doneSignal = new CountDownLatch(N);

     for (int i = 0; i &lt; N; &#43;&#43;i) // create and start threads
       new Thread(new Worker(startSignal, doneSignal)).start();

     doSomethingElse();            // don&#39;t let run yet
     startSignal.countDown();      // let all threads proceed
     doSomethingElse();
     doneSignal.await();           // wait for all to finish
   }
 }

 class Worker implements Runnable {
   private final CountDownLatch startSignal;
   private final CountDownLatch doneSignal;
   Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
      this.startSignal = startSignal;
      this.doneSignal = doneSignal;
   }
   public void run() {
      try {
        startSignal.await();
        doWork();
        doneSignal.countDown();
      } catch (InterruptedException ex) {} // return;
   }

   void doWork() { ... }
 }
```



# CyclicBarrier



## 简介

&gt; A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point. CyclicBarriers are useful in programs involving a fixed sized party of threads that must occasionally wait for each other. The barrier is called *cyclic* because it can be re-used after the waiting threads are released.

一组线程到达barrier时会被阻塞，直到最后一个线程到达barrier，被阻塞的线程才会继续执行。

与CountDownLatch的作用类似，CyclicBarrier可以执行reset方法进行重用。

## 构造方法

参数含义如下：

- **parties**：拦截的线程数量
- **barrierAction**：所有线程到达`barrier`后执行的任务

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties &lt;= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
}
```





## 成员属性

- **lock**：可重入锁，用于进行`dowait`时锁定
- **parties**：参与的线程数量
- **trip**：实际进行`await()`的`condition`
- **barrierCommand**：最后一个线程到达时执行的任务
- **count**：等待进入屏障的线程数量
- **generation**：当前的generation
  - **broken**，表示当前屏障是否被破坏。

```java
private static class Generation {
        boolean broken = false;
    }

    /** The lock for guarding barrier entry */
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
    /** The number of parties */
    private final int parties;
    /* The command to run when tripped */
    private final Runnable barrierCommand;
    /** The current generation */
    private Generation generation = new Generation();

    private int count;
```



## 核心方法



### await

可响应中断，通过调用`dowait(false, 0L)`实现

```java
public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
}
```



#### dowait

`await`的具体实现。

```java
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
			//屏障被破坏，抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
			//检查中断
            if (Thread.interrupted()) {
                //损坏屏障，唤醒所有线程
                breakBarrier();
                throw new InterruptedException();
            }
			//减少等待进入屏障的线程数量
            int index = --count;
            //index == 0表示 所有进程都已经进入
            if (index == 0) {  // tripped
                //运行的动作标识
                boolean ranAction = false;
                try {
                    //运行任务
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //进入下一代
                    nextGeneration();
                    return 0;
                } finally {
                    //如果没有改成功，损坏当前屏障
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    //如果没有设置等待时间
                    //调用condition.await()进行等待
                    if (!timed)
                        trip.await();
                    //否则调用awaitNanos()进行等待
                    else if (nanos &gt; 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //如果被中断，并且当前代的屏障没有被损坏
                    if (g == generation &amp;&amp; ! g.broken) {
                        //损坏当前屏障
                        breakBarrier();
                        throw ie;
                    } else {
                        //不是当前代，进行中断
                        Thread.currentThread().interrupt();
                    }
                }
				//检查损坏标识
                if (g.broken)
                    throw new BrokenBarrierException();
				//不等于当前代损坏表示
                if (g != generation)
                    return index;

                if (timed &amp;&amp; nanos &lt;= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```



### nextGeneration

线程进入屏障后会进行调用。

```java
private void nextGeneration() {
        // signal completion of last generation
    	//唤醒所有线程
        trip.signalAll();
        // set up next generation
    	//恢复正在等待进入屏障的线程数量
        count = parties;
        generation = new Generation();
}
```



### breakBarrier

损坏当前屏障，会唤醒所有在屏障中的线程。

```java
private void breakBarrier() {
    	//设置损坏标志
        generation.broken = true;
    	//恢复正在等待进入屏障的线程
        count = parties;
    	//唤醒所有线程
        trip.signalAll();
}
```



## 使用案例

```java
class Solver {
   final int N;
   final float[][] data;
   final CyclicBarrier barrier;

   class Worker implements Runnable {
     int myRow;
     Worker(int row) { myRow = row; }
     public void run() {
       while (!done()) {
         processRow(myRow);

         try {
           barrier.await();
         } catch (InterruptedException ex) {
           return;
         } catch (BrokenBarrierException ex) {
           return;
         }
       }
     }
   }

   public Solver(float[][] matrix) {
     data = matrix;
     N = matrix.length;
     barrier = new CyclicBarrier(N,
                                 new Runnable() {
                                   public void run() {
                                     mergeRows(...);
                                   }
                                 });
     for (int i = 0; i &lt; N; &#43;&#43;i)
       new Thread(new Worker(i)).start();

     waitUntilDone();
   }
 }
```



# Semaphore



## 简介

&gt; A counting semaphore. Conceptually, a semaphore maintains a set of permits. Each [`acquire()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Semaphore.html#acquire--) blocks if necessary until a permit is available, and then takes it. Each [`release()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Semaphore.html#release--) adds a permit, potentially releasing a blocking acquirer. However, no actual permit objects are used; the `Semaphore` just keeps a count of the number available and acts accordingly.

线程执行acquire()后，会判断permit是否可用，不可用则阻塞，可用则permit - 1。线程执行`release()`后，则permit &#43; 1，并且释放一个阻塞线程。

适用场景：控制同时访问特定资源的线程数量。保证公共资源合理使用，与OS的信号量核心理念相同。

Semaphores are often used to restrict the number of threads than can access some (physical or logical) resource. For example, here is a class that uses a semaphore to control access to a pool of items:

```java
 class Pool {
   private static final int MAX_AVAILABLE = 100;
   private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

   public Object getItem() throws InterruptedException {
     available.acquire();
     return getNextAvailableItem();
   }

   public void putItem(Object x) {
     if (markAsUnused(x))
       available.release();
   }

   // Not a particularly efficient data structure; just for demo

   protected Object[] items = ... whatever kinds of items being managed
   protected boolean[] used = new boolean[MAX_AVAILABLE];

   protected synchronized Object getNextAvailableItem() {
     for (int i = 0; i &lt; MAX_AVAILABLE; &#43;&#43;i) {
       if (!used[i]) {
          used[i] = true;
          return items[i];
       }
     }
     return null; // not reached
   }

   protected synchronized boolean markAsUnused(Object item) {
     for (int i = 0; i &lt; MAX_AVAILABLE; &#43;&#43;i) {
       if (item == items[i]) {
          if (used[i]) {
            used[i] = false;
            return true;
          } else
            return false;
       }
     }
     return false;
   }
 }
```



## 构造方法

两个构造方法：默认创建非公平策略的信号量，另一个构造方法可以选择公平策略的信号量。

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
     sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```



## 成员属性

`Semaphore`主要通过`sync`（AQS的实现类）来实现核心功能。

```java
private final Sync sync;
```



### Sync

`Sync`代码如下：

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;
		//构造方法
        Sync(int permits) {
            setState(permits);
        }
		//返回permit
        final int getPermits() {
            return getState();
        }
		//共享模式下的非公平策略获取
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining &lt; 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
		//共享模式下的释放
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current &#43; releases;
                if (next &lt; current) // overflow
                    throw new Error(&#34;Maximum permit count exceeded&#34;);
                if (compareAndSetState(current, next))
                    return true;
            }
        }
		//根据指定数量减少可用许可数量
        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next &gt; current) // underflow
                    throw new Error(&#34;Permit count underflow&#34;);
                if (compareAndSetState(current, next))
                    return;
            }
        }
		//permit不为0则更新permit，并返回permit
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }
```



### NonfairSync

非公平策略直接调用`tryAcquireShared`完成获取资源的操作。

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
```



### FairSync

公平策略中，获取共享状态时，会判断`Sync Queue`中是否有前驱元素。

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining &lt; 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
}
```



## 核心方法



### acquire

获取一个`permit`，在`permit`有效之前，将会阻塞，响应中断。

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```



### acquireUninterruptibly

不接受中断的`acquire()`.

```java
public void acquireUninterruptibly() {
    sync.acquireShared(1);
}
```



### release

释放一个`permits`。通过`AQS.releaseShared()`。

```java
public void release(int permits) {
    if (permits &lt; 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
```





# 参考

Java SE 8 Docs API

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/%E5%87%A0%E4%B8%AAjava%E5%B9%B6%E5%8F%91%E5%B7%A5%E5%85%B7%E7%B1%BB%E8%A7%A3%E6%9E%90/  

