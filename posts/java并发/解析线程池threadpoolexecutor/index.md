# 解析线程池ThreadPoolExecutor




# 什么是线程池

`Thread Pool`是一种基于池化思想管理线程的工具，经常出现在多线程程序中。

线程池的优点：

- 降低资源消耗：通过池化技术重复利用已创建线程。
- 提高响应速度：任务到达时，无需等待进程创建即可执行。
- 提高线程的可管理性：使用线程池进行统一的分配、调优和监控。
- 提供更多强大的功能：线程池具备可扩展性，允许开发人员向其中增加更多功能。



# 为什么用线程池

直接创建线程存在性能开销：

- Java中线程是基于内核线程实现的，线程的创建和销毁需要进行系统调用，性能开销较高。
- Java8中，每个`Thread`都需要有一个内核线程的支持，这意味着每个`Thread`都需要消耗一定的内核资源。Java8中每个线程栈大小是1M，Java11中，对创建线程操作进行优化，创建一个线程只需要40KB左右。
- 线程切换引起`context switch`。



​		使用线程池解决的核心问题就是**资源管理问题**，多线程环境下，不确定性会带来一些问题：频繁申请/销毁线程会带来额外的开销、存在资源耗尽的风险等。

​		使用池化思想将资源统一在一起管理的一种思想，可以最大化收益最小化风险，



# ThreadPoolExecutor

## 继承关系

`ThreadPoolExecutor`类的继承关系如下：

![ThreadPoolExecutor继承关系](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/concurrency/pool/ThreadPoolExecutor.jpg)



- Executor：顶层的`Executor`仅提供一个`execute()`接口，实现了提交任务与执行任务的解耦。
- ExecutorService：继承自`Executor`，实现了添加了其他接口，例如：
  - 为一个或一批异步任务生成Future的方法
  - 提供了管控线程池的方法，例如停止线程池运行。
- AbstractExecutorService：实现了`ExecutorService`，实现了除`execute()`以外的所有方法，将最重要的`execute()`交给`ThreadPoolExecutor`实现。



## 运行机制

`ThreadPoolExecutor`的基本运行机制如下图所示（图片来源：美团技术团队）：



![ThreadPoolExecutor](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/concurrency/pool/ThreadPoolExecutor%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

线程池内部相当于一个生产者消费者模型，将线程池分成两个部分：任务管理、线程管理。

- 任务管理相当于生产者，任务提交后，线程池判断该任务的后续操作。
  1. 直接申请线程执行该任务
  2. 存放到阻塞队列中等待
  3. 拒绝该任务。
- 线程管理部分是消费者，根据任务请求进行线程分配工作，当线程执行完任务后会继续获取新的任务去执行，最终当线程获取不到任务时，线程会进行回收。



## 构造方法



核心的构造方法如下，主要参数有：

- **corePoolSize**：核心线程数量
- **maximumPoolSize**：最大线程数量
- **workQueue**：BlockingQueue类型，保存等待执行任务的阻塞队列，当提交一个新的任务到线程池时，线程池根据当前状态决定后续处理。可选择以下几种：
  - ArrayBlockingQueue
  - LinkedBlockingQueue：Executors.newFixedThreadPool使用该队列
  - SynchronousQueue：同步队列，容量为0，put必须等待take，take等待put，Executors.newCachedThreadPool使用该队列。

- **keepAliveTime**：线程池维护线程所允许的时间。当线程池中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了keepAliveTime。
- **threadFactory**：它是`ThreadFactory`类型的变量，用来创建新线程。
- **handler**：`RejectedExecutionHandler`类型，表示线程池的拒绝策略。如果阻塞队列满了并且没有空闲的线程，这时如果继续提交任务，就需要采取一种策略处理该任务。

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue&lt;Runnable&gt; workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize &lt; 0 ||
            maximumPoolSize &lt;= 0 ||
            maximumPoolSize &lt; corePoolSize ||
            keepAliveTime &lt; 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```





## 线程池的状态



线程池的运行状态，由`AtomicInteger ctl`维护，其中分为两个参数：`runState`和`workerCount`，高3位存储`runState`，低29位存储`workerCount`，提供了位运算的方法来获取对应的参数。线程池的运行状态，通过内部进行调整。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 &lt;&lt; COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 &lt;&lt; COUNT_BITS;
private static final int SHUTDOWN   =  0 &lt;&lt; COUNT_BITS;
private static final int STOP       =  1 &lt;&lt; COUNT_BITS;
private static final int TIDYING    =  2 &lt;&lt; COUNT_BITS;
private static final int TERMINATED =  3 &lt;&lt; COUNT_BITS;

//获取运行状态
private static int runStateOf(int c)     { return c &amp; ~CAPACITY; }
//获取活动线程数
private static int workerCountOf(int c)  { return c &amp; CAPACITY; }
//获取运行状态和活动线程数
private static int ctlOf(int rs, int wc) { return rs | wc; }
```



线程池状态表如下：

|  运行状态  |                           状态描述                           |
| :--------: | :----------------------------------------------------------: |
|  RUNNING   |        能接受新提交的任务，并且能处理阻塞队列中的任务        |
|  SHUTDOWN  | 关闭状态，不再接受新提交的任务，但是可以继续处理阻塞队列中的任务。 |
|    STOP    | 不能接受新的任务，也不处理队列中的任务，会中断正在处理任务的线程。 |
|  TIDYING   | 如果所有的任务都终止了，`workerCount`位0，线程池会调用`terminated()`进入TERMINATED状态。 |
| TERMINATED |            执行完`terminated()`方法后进入该状态。            |



线程池的状态转换过程如下图所示：

![线程池状态转换过程](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/concurrency/pool/pool-status.jpg)





## 任务调度机制



用户提交一个任务给线程池，线程池如何进行调度，如何对任务进行管理。这是这部分的核心问题。

调度工作是由`execute`方法完成，由此可以看出该方法的重要性。

其基本执行过程如下：

1. 首先检查线程池运行状态，如果不是`RUNNING`，则直接拒绝。
2. 如果workerCount &lt; corePoolSize，则创建并启动一个线程来执行新任务。
3. 如果workerCount &gt;= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到阻塞队列中。
4. 如果workerCount &gt;= corePoolSize &amp;&amp; workCount &lt; maximumPoolSize，且线程池内阻塞队列已满，则创建并启动一个线程来执行新任务
5. 如果workerCount &gt;= maximumPoolSize，并且线程池内的阻塞队列已满，则根据拒绝策略来处理该任务。

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
    	//workerCount &lt; corePoolSize
        if (workerCountOf(c) &lt; corePoolSize) {
            //创建一个新的线程放入线程池，并将任务加到该线程。
            if (addWorker(command, true))
                return;
            //如果添加失败，更新c
            c = ctl.get();
        }
    	//判断当前线程池是RUNNING状态，并且将任务添加到workQueue成功
        if (isRunning(c) &amp;&amp; workQueue.offer(command)) {
            //重新获取ctl的值
            int recheck = ctl.get();
            //再次判断线程池的状态，如果不是运行状态，将任务移出队列成功后，进行拒绝
            if (! isRunning(recheck) &amp;&amp; remove(command))
                reject(command);
            //如果workCount == 0，则添加新线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
    	//1. 线程不是RUNNING状态 2.线程池是RUNNING状态，但是workerCount &gt;= corePoolSize &amp;&amp; workerCount已满
        else if (!addWorker(command, false))
            //失败则进行拒绝
            reject(command);
    }
```



其执行流程如下图所示：

![任务调度流程](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/execute.png)



### addWorker

该方法主要是在线程池中新建一个`worker`线程并执行任务，两个参数的含义如下：

- `firstTask`：执行新增的线程执行的第一个任务
- `core`：检测标识
  - `true`：新增线程时会判断当前活动线程是否小于`corePoolSize`
  - `false`：新增线程时会判断当前活动线程是否少于`maximumPoolSize`

方法的具体含义参考代码注释。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            //获取线程池运行状态
            int c = ctl.get();
            int rs = runStateOf(c);

            //如果rs &gt;= SHUTDOWN，则表示此时不再接收新任务
            //再检查三个条件，其中一个不满足则添加失败
            //1. rs == SHUTDOWN：关闭状态则不接受新任务
            //2. firstTask为空
            //3. 阻塞队列不为空
            //队列中没有任务则不需要再添加线程
            if (rs &gt;= SHUTDOWN &amp;&amp;
                ! (rs == SHUTDOWN &amp;&amp;
                   firstTask == null &amp;&amp;
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                //获取线程数量
                int wc = workerCountOf(c);
                //检查线程数量是否超过CAPACITY（二进制为29个1）
                //根据传入的core判断与哪个参数进行对比
                if (wc &gt;= CAPACITY ||
                    wc &gt;= (core ? corePoolSize : maximumPoolSize))
                    //如果条件成立，则拒绝创建线程
                    return false;
                //CAS尝试增加workerCount，成功则跳出外层循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //当前运行状态不等于rs，说明状态已改变，则返回内层循环继续执行
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //根据firstTask创建Worker对象
            w = new Worker(firstTask);
            //获取worker的thread
            final Thread t = w.thread;
            if (t != null) {
                //上一个可重入锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
					//rs &lt; SHUTDOWN表示是RUNNING状态
                    //或者是RUNING状态并且firstTask为null
                    if (rs &lt; SHUTDOWN ||
                        (rs == SHUTDOWN &amp;&amp; firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //将worker加入工作线程集合中。
                        workers.add(w);
                        int s = workers.size();
                        // largestPoolSize记录着线程池中出现过的最大线程数量
                        //如果添加之后的工作线程集合size &gt; largestPoolSize
                        if (s &gt; largestPoolSize)
                            //更新线程池中出现的最大线程数量
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //添加成功，启动线程执行任务
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```





### 拒绝策略

线程池提供了如下四种策略：

1. **AbortPolicy**：直接抛出异常，这是默认策略。
2. **CallerRunsPolicy**：用调用者所在的线程来执行任务。
3. **DiscardOldestPolicy**：丢弃阻塞队列中靠最前的任务，并执行当前任务。
4. **DiscardPolicy**：直接丢弃任务。



```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
        
        public CallerRunsPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    public static class AbortPolicy implements RejectedExecutionHandler {

        public AbortPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException(&#34;Task &#34; &#43; r.toString() &#43;
                                                 &#34; rejected from &#34; &#43;
                                                 e.toString());
        }
    }

    public static class DiscardPolicy implements RejectedExecutionHandler {

        public DiscardPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

    public static class DiscardOldestPolicy implements RejectedExecutionHandler {

        public DiscardOldestPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```



## 任务阻塞机制

任务阻塞机制是线程池管理任务的核心机制。线程池将任务和线程进行解耦，以生产者消费者模式，通过一个阻塞队列来实现的。任务缓存到阻塞队列，线程从阻塞队列中获取任务。



## Worker

&gt;  Q：线程池如何管理进程？

线程池中每一个线程被封装成`Worker`对象，通过对`Worker`对象的管理，从而达到对进程管理的目的。下面这个`HashSet`存储的是`Worker`集合。

```java
private final HashSet&lt;Worker&gt; workers = new HashSet&lt;Worker&gt;();
```

`Worker`继承了`AQS`，使用`AQS`实现了独占锁的功能，从`tryAcquire`可以看出禁止重入。其主要含义如下：

- 独占状态：表示当前线程正在执行任务，则不应该中断线程。
- 空闲状态：没有在处理任务，可以对线程中断。



其它注意点：

- 线程池在执行`shutdown`方法或`tryTerminate`方法时会调用`interruptIdleWorkers`方法来中断空闲的线程，`interruptIdleWorkers`方法会使用`tryLock`方法来判断线程池中的线程是否是空闲状态；
- 设置成不可重入的原因：不希望任务在运行时重新获得锁，从而调用一些会中断运行时线程的方法。

`Worker`代码如下：

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

       //worker持有的线程
        final Thread thread;
       //初始化的任务，可以为null
        Runnable firstTask;
        //任务计数器
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            //AQS默认为0，设置为-1是为了禁止执行任务前对线程进行中断
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            //通过ThreadFactory创建线程
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
		//不可重入
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() &gt;= 0 &amp;&amp; (t = thread) != null &amp;&amp; !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```



### runWorker

在`Worker`类中的`run()`调用该方法来执行任务，主要逻辑如下：

1. while循环通过`getTask()`获取任务。
2. 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态（因为需要继续运行）。
3. 调用`task.run()`运行任务
4. 如果阻塞队列中没有任务可执行则跳出循环，执行`processWorkerExit()`回收线程。



其代码如下：

```java
final void runWorker(Worker w) {
    	//获取线程
        Thread wt = Thread.currentThread();
    	//获取第一个任务
        Runnable task = w.firstTask;
        w.firstTask = null;
    	//允许中断
        w.unlock();
    	//是否因为异常退出循环
        boolean completedAbruptly = true;
        try {
            //task为空，则通过getTask来获取任务
            while (task != null || (task = getTask()) != null) {
                //禁止中断
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                //如果线程池正在停止，保证当前线程是中断状态
                //如果不是，确保当前线程不是中断状态
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &amp;&amp;
                      runStateAtLeast(ctl.get(), STOP))) &amp;&amp;
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks&#43;&#43;;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```



### getTask

该方法在runWorker中被调用，用于取出workQueue中的任务，代码与注释如下：

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
		//死循环获取队列任务
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
			//判断线程池状态
            // Check if queue empty only if necessary.
            if (rs &gt;= SHUTDOWN &amp;&amp; (rs &gt;= STOP || workQueue.isEmpty())) {
                //若已经停止，控制工作线程数量，并return null
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc &gt; corePoolSize;
			//控制线程数量
            if ((wc &gt; maximumPoolSize || (timed &amp;&amp; timedOut))
                &amp;&amp; (wc &gt; 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //取出任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```





# Reference

https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/%E8%A7%A3%E6%9E%90%E7%BA%BF%E7%A8%8B%E6%B1%A0threadpoolexecutor/  

