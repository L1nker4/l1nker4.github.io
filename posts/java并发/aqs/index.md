# AQS原理与源码分析


# 简介

队列同步器AbstractQueuedSynchronizer，是用来构建锁或者其他同步组键的基础框架，它使用了一个int成员变量**state**表示同步状态，通过CLH队列完成获取资源的线程排队工作。

AQS的主要使用方式是继承，字类通过继承同步器并实现它的抽象方法来管理同步状态。AQS本身只是定义若干同步状态获取和释放的方法提供给字类来实现。

锁是面向使用者的，它定义了使用者与锁交互的接口，隐藏了实现细节。AQS是面向锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理，线程的排队，等待与唤醒等底层操作。

AQS定义了两种资源共享的方式：

- Exclusive（独占）：只有一个线程能执行，如ReentrantLock，其中又可分为公平锁和非公平锁：
  - 公平锁：线程按照队列的顺序获取锁
  - 非公平锁：线程无视顺序，去抢锁
- Share（共享）：多个线程可同时执行，如Semaphore/CountDownLatch。


AQS的设计是基于模板方法模式的，使用者继承AbstractQueuedSynchronizer并重写指定方法，重写的方法是对同步状态`state`的获取释放等操作。


可重写方法如下，arg参数为获取锁的次数。

|                    名称                     |                             描述                             |
| :-----------------------------------------: | :----------------------------------------------------------: |
|    protected boolean tryAcquire(int arg)    | 独占方式，尝试获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后通过CAS设置同步状态，成功返回true，失败返回false |
|    protected boolean tryRelease(int arg)    |  独占方式，尝试释放同步状态，成功返回true，失败则返回false   |
|   protected int tryAcquireShared(int arg)   | 共享方式，尝试获取同步状态，返回0表示成功，但是没有剩余可用资源，负数表示失败，正数表示成功，并且有剩余资源。 |
| protected boolean tryReleaseShared(int arg) |   共享方式，尝试释放同步状态，成功返回true，失败返回false    |
|    protected boolean isHeldExclusively()    |                 判断当前线程是否正在独占资源                 |



模板方法：

|                         方法名称                          |                             描述                             |
| :-------------------------------------------------------: | :----------------------------------------------------------: |
|                   void acquire(int arg)                   | 独占锁获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法会调用重写的tryAcquire()方法 |
|            void acquireInterruptibly(int arg)             | 与acquire相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列，如果当前线程被中断，该方法会抛出`InterruptedException`并返回。 |
|    boolean tryAcquireNanos(int arg, long nanosTimeout)    | 在acquireInterruptibly的基础上增加了超时限制，如果当前线程在超时时间之内没有获取同步状态，那么将会返回false，获取到了返回true |
|                void acquireShared(int arg)                | 共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占锁获取的主要区别式同一时刻可以有多个线程获取同步状态 |
|         void acquireSharedInterruptibly(int arg)          |                与acquireShared相同，响应中断                 |
| boolean tryAcquireSharedNanos(int arg, long nanosTimeout) |                         加了超时限制                         |
|                 boolean release(int arg)                  | 独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中的第一个节点线程唤醒 |
|              boolean releaseShared(int arg)               |                     共享式的释放同步状态                     |
|           Collection&lt;Thread&gt; getQueuedThreads()           |                获取等待在同步队列上的线程集合                |

模板方法基本分成3类：独占式获取与释放，共享式获取与释放，查询同步队列中的情况。



AQS整体方法架构可以参照下图（来源：美团技术团队）

![AQS方法架构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/concurrency/aqs/82077ccf14127a87b77cefd1ccf562d3253591.png)



# 原理

核心思想：如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并将共享资源设置为锁定状态，如果被请求的共享资源被占用，就将请求资源的线程加入CLH队列中。

CLH：Craig、Landin and Hagersten队列，是单向链表，AQS中的队列是CLH变体的双向队列（FIFO），每一个节点都是等待资源的线程。

AQS使用一个volatile修饰的int类型的成员变量`state`来表示同步状态，通过内置的FIFO队列来完成资源获取的排队工作，通过CAS方式完成对`state`的修改。


## 类的继承关系

`AbstractQueuedSynchronizer`继承自`AbstractOwnableSynchronizer`，并且实现`Serializable`接口。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
```

`AbstractOwnableSynchronizer`抽象类可以设置独占资源线程和获取独占资源的线程。

```java
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
    
    // 版本序列号
    private static final long serialVersionUID = 3737899427754241961L;
    // 构造方法
    protected AbstractOwnableSynchronizer() { }
    // 独占模式下的线程
    private transient Thread exclusiveOwnerThread;
    
    // 设置独占线程 
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    
    // 获取独占线程 
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

## Node

每一个阻塞的线程都会被封装成一个Node节点，放入Sync Queue。

```java
static final class Node {
        //线程节点的两种状态，独享模式和共享模式
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;

        //表示当前节点已取消调度
        static final int CANCELLED =  1;
        //表示后继节点在等待当前节点唤醒，后继节点入队时，会见前继节点状态更新为SIGNAL
        static final int SIGNAL    = -1;
        //表示节点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取锁。
        static final int CONDITION = -2;
        //SHARED模式下，前继节点不仅会唤醒后继节点，也可能唤醒后继的后继节点
        static final int PROPAGATE = -3;

        //当前节点的状态
        volatile int waitStatus;

        //前继节点
        volatile Node prev;

        //后继节点
        volatile Node next;

        //处于当前节点的线程
        volatile Thread thread;

        //指向下一个处于CONDITION状态的节点
        Node nextWaiter;

        //判断是否是SHARED状态
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        //返回前继节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

## ConditionObject

该类实现了`Condition`接口，该接口定义了如下规范：

```java
public interface Condition {

    // 等待，当前线程在接到信号或被中断之前一直处于等待状态
    void await() throws InterruptedException;
    
    // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
    void awaitUninterruptibly();
    
    //等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态 
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    
    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法在行为上等效于: awaitNanos(unit.toNanos(time)) &gt; 0
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    
    // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
    boolean awaitUntil(Date deadline) throws InterruptedException;
    
    // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
    void signal();
    
    // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。
    void signalAll();
}
```

```java
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        //condition队列的头节点
        private transient Node firstWaiter;
        //condition队列的尾节点
        private transient Node lastWaiter;

        //构造方法
        public ConditionObject() { }

        // Internal methods

        //添加新的waiter到wait队列
        private Node addConditionWaiter() {
            //保存尾节点
            Node t = lastWaiter;
            //尾节点不为空，并且尾节点的状态不为CONDITION
            if (t != null &amp;&amp; t.waitStatus != Node.CONDITION) {
                //清除状态为CONDITION的结点
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            //将当前线程设置为node，状态为CONDITION
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)//尾节点为空
                //设置头节点为node
                firstWaiter = node;
            else
                //设置尾节点的next指向node
                t.nextWaiter = node;
            //更新condition队列的尾节点
            lastWaiter = node;
            return node;
        }

        /**
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &amp;&amp;
                     (first = firstWaiter) != null);
        }

        /**
         * Removes and transfers all nodes.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }

        /**
         * Unlinks cancelled waiter nodes from condition queue.
         * Called only while holding lock. This is called when
         * cancellation occurred during condition wait, and upon
         * insertion of a new waiter when lastWaiter is seen to have
         * been cancelled. This method is needed to avoid garbage
         * retention in the absence of signals. So even though it may
         * require a full traversal, it comes into play only when
         * timeouts or cancellations occur in the absence of
         * signals. It traverses all nodes rather than stopping at a
         * particular target to unlink all pointers to garbage nodes
         * without requiring many re-traversals during cancellation
         * storms.
         */
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }

        // public methods

        /**
         * Moves the longest-waiting thread, if one exists, from the
         * wait queue for this condition to the wait queue for the
         * owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        /**
         * Moves all threads from the wait queue for this condition to
         * the wait queue for the owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }

        /**
         * Implements uninterruptible condition wait.
         * &lt;ol&gt;
         * &lt;li&gt; Save lock state returned by {@link #getState}.
         * &lt;li&gt; Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * &lt;li&gt; Block until signalled.
         * &lt;li&gt; Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * &lt;/ol&gt;
         */
        public final void awaitUninterruptibly() {
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if (Thread.interrupted())
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted)
                selfInterrupt();
        }

        /*
         * For interruptible waits, we need to track whether to throw
         * InterruptedException, if interrupted while blocked on
         * condition, versus reinterrupt current thread, if
         * interrupted while blocked waiting to re-acquire.
         */

        /** Mode meaning to reinterrupt on exit from wait */
        private static final int REINTERRUPT =  1;
        /** Mode meaning to throw InterruptedException on exit from wait */
        private static final int THROW_IE    = -1;

        /**
         * Checks for interrupt, returning THROW_IE if interrupted
         * before signalled, REINTERRUPT if after signalled, or
         * 0 if not interrupted.
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }

        /**
         * Throws InterruptedException, reinterrupts current thread, or
         * does nothing, depending on mode.
         */
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

        /**
         * Implements interruptible condition wait.
         * &lt;ol&gt;
         * &lt;li&gt; If current thread is interrupted, throw InterruptedException.
         * &lt;li&gt; Save lock state returned by {@link #getState}.
         * &lt;li&gt; Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * &lt;li&gt; Block until signalled or interrupted.
         * &lt;li&gt; Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * &lt;li&gt; If interrupted while blocked in step 4, throw InterruptedException.
         * &lt;/ol&gt;
         */
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) &amp;&amp; interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        /**
         * Implements timed condition wait.
         * &lt;ol&gt;
         * &lt;li&gt; If current thread is interrupted, throw InterruptedException.
         * &lt;li&gt; Save lock state returned by {@link #getState}.
         * &lt;li&gt; Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * &lt;li&gt; Block until signalled, interrupted, or timed out.
         * &lt;li&gt; Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * &lt;li&gt; If interrupted while blocked in step 4, throw InterruptedException.
         * &lt;/ol&gt;
         */
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() &#43; nanosTimeout;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout &lt;= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout &gt;= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) &amp;&amp; interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }

        /**
         * Implements absolute timed condition wait.
         * &lt;ol&gt;
         * &lt;li&gt; If current thread is interrupted, throw InterruptedException.
         * &lt;li&gt; Save lock state returned by {@link #getState}.
         * &lt;li&gt; Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * &lt;li&gt; Block until signalled, interrupted, or timed out.
         * &lt;li&gt; Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * &lt;li&gt; If interrupted while blocked in step 4, throw InterruptedException.
         * &lt;li&gt; If timed out while blocked in step 4, return false, else true.
         * &lt;/ol&gt;
         */
        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() &gt; abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) &amp;&amp; interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        /**
         * Implements timed condition wait.
         * &lt;ol&gt;
         * &lt;li&gt; If current thread is interrupted, throw InterruptedException.
         * &lt;li&gt; Save lock state returned by {@link #getState}.
         * &lt;li&gt; Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * &lt;li&gt; Block until signalled, interrupted, or timed out.
         * &lt;li&gt; Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * &lt;li&gt; If interrupted while blocked in step 4, throw InterruptedException.
         * &lt;li&gt; If timed out while blocked in step 4, return false, else true.
         * &lt;/ol&gt;
         */
        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            long nanosTimeout = unit.toNanos(time);
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() &#43; nanosTimeout;
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout &lt;= 0L) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout &gt;= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) &amp;&amp; interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        //  support for instrumentation

        /**
         * Returns true if this condition was created by the given
         * synchronization object.
         *
         * @return {@code true} if owned
         */
        final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
            return sync == AbstractQueuedSynchronizer.this;
        }

        /**
         * Queries whether any threads are waiting on this condition.
         * Implements {@link AbstractQueuedSynchronizer#hasWaiters(ConditionObject)}.
         *
         * @return {@code true} if there are any waiting threads
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        protected final boolean hasWaiters() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    return true;
            }
            return false;
        }

        /**
         * Returns an estimate of the number of threads waiting on
         * this condition.
         * Implements {@link AbstractQueuedSynchronizer#getWaitQueueLength(ConditionObject)}.
         *
         * @return the estimated number of waiting threads
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        protected final int getWaitQueueLength() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int n = 0;
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    &#43;&#43;n;
            }
            return n;
        }

        /**
         * Returns a collection containing those threads that may be
         * waiting on this Condition.
         * Implements {@link AbstractQueuedSynchronizer#getWaitingThreads(ConditionObject)}.
         *
         * @return the collection of threads
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        protected final Collection&lt;Thread&gt; getWaitingThreads() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            ArrayList&lt;Thread&gt; list = new ArrayList&lt;Thread&gt;();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION) {
                    Thread t = w.thread;
                    if (t != null)
                        list.add(t);
                }
            }
            return list;
        }
    }
```

## State

AQS中有一个state字段，为同步状态，用volatile修饰。AQS中提供了几个访问该字段的方法：

```java
	//返回当前state
    protected final int getState() {
        return state;
    }
	//设置state
	protected final void setState(int newState) {
        state = newState;
    }
	//CAS方式更新state
	protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

可以通过修改`state`字段来实现独占模式和共享模式。

- 独占模式下只能有一个线程进入。
  1. 初始化`state`为0
  2. 线程A申请独占操作
  3. 判断`state`是否为0
  4. 如果不为0，则线程A阻塞
  5. 为0则设置`state`为1，表示独占
- 共享模式下可以有多个线程进入
  1. 初始化`state = n`
  2. 线程A,B,C,D进行共享操作
  3. 判断`state`是否大于0
  4. 不大于0则线程阻塞
  5. 大于0则进行CAS自减



## 类的属性

```java
    //头节点
    private transient volatile Node head;

    尾节点
    private transient volatile Node tail;

    //state
    private volatile int state;
    
    //unsafe
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //通过内存偏移地址来修改变量值
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;
    
    static final long spinForTimeoutThreshold = 1000L;
    //获取各个变量的内存偏移地址
    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField(&#34;state&#34;));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField(&#34;head&#34;));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField(&#34;tail&#34;));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField(&#34;waitStatus&#34;));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField(&#34;next&#34;));

        } catch (Exception ex) { throw new Error(ex); }
    }
    

```

## 构造方法

`protected`修饰，供子类调用。

```java
protected AbstractQueuedSynchronizer() { }
```


## 方法



### acquire

独占模式下获取共享资源，如果当前线程获取共享资源成功，则由该方法返回，否则，将会进入同步队列等待，直到获取资源为止，整个过程忽略中断。

```java
	public final void acquire(int arg) {
        if (!tryAcquire(arg) &amp;&amp;
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```


#### tryAcquire

尝试去获取独占资源，如果获取成功，直接返回true，否则返回false。

```java
	protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```

在AQS中只是定义一个接口，具体的资源获取和释放方式交给自定义的同步器去实现。



#### addWaiter

此方法将当前线程加到队尾，并返回当前线程所在的节点。

```java
	private Node addWaiter(Node mode) {
        //将当前线程和模式构造成节点
        Node node = new Node(Thread.currentThread(), mode);
        //pred指向尾节点tail
        Node pred = tail;
        if (pred != null) {
            //新构造的节点加入队尾
            node.prev = pred;
            //比较pred是否为尾节点，是则将尾节点设置为node
            if (compareAndSetTail(pred, node)) {
                //设置尾节点的next
                pred.next = node;
                return node;
            }
        }
        //如果队列为空，使用enq方法入队
        enq(node);
        return node;
    }
```



#### enq

`enq`使用自旋方式来确保节点的插入

```java
private Node enq(final Node node) {
    //CAS自旋，直到成功加入队尾
    for (;;) {
        Node t = tail;
        if (t == null) { 
            //队列为空时，创建一个空节点作为head节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //尾节点不为空时，将node节点的prev连接到t
            node.prev = t;
            //比较节点t是否为尾节点，若是则将尾节点设置为node
            if (compareAndSetTail(t, node)) {
                //设置尾节点的next指向node
                t.next = node;
                return t;
            }
        }
    }
}
```





#### acquireQueued

如果执行到此方法，说明该线程获取资源失败，已被放入队列尾部。acquireQueued方法具体流程如下：

1. 节点进入队尾后，判断如果前驱节点是头节点就尝试获取资源，如果成功，直接返回
2. 否则就通过shouldParkAfterFailedAcquire判断前驱节点状态是否为SIGNAL，是则park当前节点，否则不进行park操作。
3. 如果park了当前线程，之后某个线程对本线程的unpark后，本线程会被唤醒，将

```java
	final boolean acquireQueued(final Node node, int arg) {
        //标记是否成功拿到锁
        boolean failed = true;
        try {
            //标记是否被中断
            boolean interrupted = false;
            //自旋
            for (;;) {
                //定义p为该节点的前驱节点
                final Node p = node.predecessor();
                //如果前驱节点是head，并且成功获得锁
                if (p == head &amp;&amp; tryAcquire(arg)) {
                    //将头结点设置为当前节点
                    setHead(node);
                    p.next = null; // help GC
                    //成功获取锁
                    failed = false;
                    //返回等待过程中是否被中断过
                    return interrupted;
                }
                //获取资源失败就通过shouldParkAfterFailedAcquire方法判断节点状态是否为SIGNAL
                //如果是SIGNAL状态，执行parkAndCheckInterrupt方法挂起线程，如果被唤醒，检查是否被中断
                if (shouldParkAfterFailedAcquire(p, node) &amp;&amp;
                    parkAndCheckInterrupt())
                    //是中断的话，将中断标志设置为true
                    interrupted = true;
            }
        } finally {
            //如果获取资源失败，就取消节点在队列中的等待
            if (failed)
                cancelAcquire(node);
        }
    }
```



##### shouldParkAfterFailedAcquire

此方法用于检查状态，检查是否进入SIGNAL状态。只有当前节点的前驱节点的状态为`SIGNAL`时，才对该节点内部线程进行`park`操作。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	//定义pred节点的状态
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            //表示pred节点处于SIGNAL状态，可以进行park操作
            return true;
        if (ws &gt; 0) {
            //CANCELLED状态，表示获取锁的请求取消
            do {
                //如果前驱节点放弃了请求，就一直往前找到正常等待状态的节点
                node.prev = pred = pred.prev;
            } while (pred.waitStatus &gt; 0);
            //改变pred的next域
            pred.next = node;
        } else {
            //如果前驱节点正常，就把前驱节点地状态设置为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        //不能进行park操作
        return false;
    }
```



##### parkAndCheckInterrupt

此方法主要用于挂起当前线程，并返回中断标志。

```java
private final boolean parkAndCheckInterrupt() {
    	//调用park方法使线程进入waiting状态
        LockSupport.park(this);
    	//如果被唤醒，检查是否是被中断，并清除中断标记位
        return Thread.interrupted();
    }
```



#### cancelAcquire

acquireQueued方法中，获取资源失败执行的方法。主要功能就是取消当前线程对资源的获取，即设置该节点的状态为CANCELLED。

```java
private void cancelAcquire(Node node) {
        //过滤空节点
        if (node == null)
            return;
		//将该节点中保存的线程信息删除
        node.thread = null;
    	//定义pred节点为node的前驱节点
        Node pred = node.prev;
    	//通过前驱节点找到不为CANCELLED状态的节点
        while (pred.waitStatus &gt; 0)
            node.prev = pred = pred.prev;

        //过滤后的前驱节点的后继节点
        Node predNext = pred.next;
       	//将node状态设置为CANCELLED
        node.waitStatus = Node.CANCELLED;
        //如果node节点是尾节点，则设置尾节点是pred节点
        if (node == tail &amp;&amp; compareAndSetTail(node, pred)) {
            //将tail的后继节点设置为null
            compareAndSetNext(pred, predNext, null);
        } else {
            //node节点不为尾节点，或者compareAndSet失败
            int ws;
            //如果pred不是头节点
            //判断状态是否为SIGNAL，不是的话，将节点状态设置为SIGNAL看是否成功
            //判断当前节点的线程是否为null
            if (pred != head &amp;&amp;
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws &lt;= 0 &amp;&amp; compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &amp;&amp;
                pred.thread != null) {
                //当前节点的前驱节点的后继指针指向当前节点的后继节点
                Node next = node.next;
                if (next != null &amp;&amp; next.waitStatus &lt;= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //上述条件不满足，那就唤醒当前节点的后继节点
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```



#### acquire小结

具体流程：

1. 调用自定义同步器的tryAcquire()方法尝试直接获取资源，如果成功直接返回。
2. 没有成功就将线程加入等待队列尾部，并标记为独占状态。
3. acquireQueued()使在等待队列挂起，有机会（被unpark）会去尝试获取资源，获取到资源直接返回，如果这个过程被中断，就返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的，只有获取资源后自我中断selfInterrupt()。

acquire的流程也就是`ReentrantLock.lock()`方法的流程。通过调用`acquire(1);`实现。





### release

独占模式下释放共享资源，如果释放资源成功（state = 0），它会唤醒同步队列中第一个节点，这也是`unlock()`的语义。 

```java
	public final boolean release(int arg) {
        //调用tryRelease
        if (tryRelease(arg)) {
            //头节点
            Node h = head;
            if (h != null &amp;&amp; h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```



#### tryRelease

和`tryAcquire()`一样，这个方法需要自定义同步器实现。此方法尝试去释放资源

```java
	protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```





#### unparkSuccessor

此方法用于唤醒队列中最前面的非CANCELED状态的线程。

```java
	private void unparkSuccessor(Node node) {
        //判断节点的状态是否为非CANCELLED状态
        int ws = node.waitStatus;
        if (ws &lt; 0)
            //如果是非CANCELLED状态，将状态设置为0
            compareAndSetWaitStatus(node, ws, 0);
		//定义s为node的后继节点
        Node s = node.next;
        //判断s是否为空节点或者是否为CANCELLED状态
        if (s == null || s.waitStatus &gt; 0) {
            s = null;
            //从尾节点往前找到最前面那个为非CANCELLED状态的线程
            for (Node t = tail; t != null &amp;&amp; t != node; t = t.prev)
                if (t.waitStatus &lt;= 0)
                    s = t;
        }
        //如果该节点不为空，就unpark当前节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```



#### release小结

release()在独占模式下释放资源。如果release时出现异常，没有unpark队列中的其他节点。会导致线程永远挂起，无法被唤醒。





### acquireShared

共享模式的获取共享资源的入口，如果当前线程未获取到共享资源，将会进入同步队列等待。  

```java
	public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) &lt; 0)
            doAcquireShared(arg);
    }
```

流程：

1. tryAcquireShared()尝试获取资源，成功则直接返回；

2. 失败则通过doAcquireShared()进入等待队列，直到获取到资源为止才返回。



#### tryAcquireShared

tryAcquireShared由自定义同步器实现。在acquireShared方法中，已经将返回值的语义定义好了，负值表示获取失败，0代表获取成功，但是没有剩余资源，正数表示获取成功，还有剩余资源，其它线程还可以获取。

```java
	protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }
```



#### doAcquireShared

此方法将当前线程加入等待队列尾部进行休息，直到其他线程释放资源唤醒自己。自己拿到资源后才返回。

```java
	private void doAcquireShared(int arg) {
        //加入队列尾部
        final Node node = addWaiter(Node.SHARED);
        //是否获取资源成功标记
        boolean failed = true;
        try {
            //是否被中断标记
            boolean interrupted = false;
            for (;;) {
                //前驱节点
                final Node p = node.predecessor();
                //如果前驱节点是头节点
                if (p == head) {
                    //尝试获取资源
                    int r = tryAcquireShared(arg);
                    if (r &gt;= 0) {
                        //获取成功，将head指向node节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        //如果等待过程中被中断
                        if (interrupted)
                            //自我中断
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //进入park状态，等待被unpark
                if (shouldParkAfterFailedAcquire(p, node) &amp;&amp;
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```



#### setHeadAndPropagate

```java
	private void setHeadAndPropagate(Node node, int propagate) {
        //保存老的头节点
        Node h = head;
        //将头节点指向自己
        setHead(node);
        //传进来的propagate为线程执行tryAcquireShared的返回值
        //大于0代表获取资源成功，并且还有剩余资源
        if (propagate &gt; 0 || h == null || h.waitStatus &lt; 0 ||
            (h = head) == null || h.waitStatus &lt; 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```



### releaseShared

共享模式下的线程释放共享资源的顶层入口。释放掉资源，唤醒后继节点。

```java
	public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```



#### doReleaseShared

此方法用于唤醒后继节点。

```java
	private void doReleaseShared() {
        
        for (;;) {
            //保存头节点
            Node h = head;
            //如果头节点不为空，并且不是尾节点
            if (h != null &amp;&amp; h != tail) {
                int ws = h.waitStatus;
                //判断头节点的线程状态是否为SIGNAL
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;
                    //唤醒后继节点
                    unparkSuccessor(h);
                }
                //不是SIGNAL，就继续自旋
                else if (ws == 0 &amp;&amp;
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;               
            }
            //如果是头节点就直接跳出
            if (h == head)
                break;
        }
    }
```





## 应用

AQS作为并发编程的底层框架，为其它很多同步工具提供了很多应用场景。大致如表所述：

|        同步工具        | 与AQS的关联                                                  |
| :--------------------: | ------------------------------------------------------------ |
|     ReentrantLock      | 使用AQS保存锁重复持有的次数。当一个线程获取锁时，ReentrantLock记录当前获得锁的线程标识，用于检测是否重复获取，以及错误线程试图解锁操作时异常情况的处理。 |
|       Semaphore        | 使用AQS同步状态来保存信号量的当前计数。tryRelease会增加计数，acquireShared会减少计数。 |
|     CountDownLatch     | 使用AQS同步状态来表示计数。计数为0时，所有的Acquire操作（CountDownLatch的await方法）才可以通过。 |
| ReentrantReadWriteLock | 使用AQS同步状态中的16位保存写锁持有的次数，剩下的16位用于保存读锁的持有次数。 |
|   ThreadPoolExecutor   | Worker利用AQS同步状态实现对独占线程变量的设置（tryAcquire和tryRelease）。 |





## 参考

[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

《Java并发编程的艺术》

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/aqs/  

