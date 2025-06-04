# Java线程基础知识


## 进程与线程



### 什么是进程

操作系统在运行一个程序时，会为其创建一个进程，操作系统调度的最小单元是线程，也叫轻量级进程，在一个进程里可以创建多个线程，多个线程共享进程的堆和方法区两块内存空间。



#### 进程和线程的区别

- **进程是操作系统进行资源分配的基本单位，而线程是操作系统进行调度的基本单位。**

- 进程单独占有一定的内存地址空间，所以进程间存在内存隔离，数据是分开的，数据共享复杂但是同步简单，各个进程之间互不干扰；而线程共享所属进程占有的内存地址空间和资源，数据共享简单，但是同步复杂。
- 进程单独占有一定的内存地址空间，一个进程出现问题不会影响其他进程，不影响主程序的稳定性，可靠性高；一个线程崩溃可能影响整个程序的稳定性，可靠性较低。
- 进程单独占有一定的内存地址空间，进程的创建和销毁不仅需要保存寄存器和栈信息，还需要资源的分配回收以及页调度，开销较大；线程只需要保存寄存器和栈信息，开销较小。





#### 上下文切换

上下文切换是指CPU从一个进程（线程）切换到另一个进程（线程）。**上下文是指某一个时间点CPU寄存器和PC的数据**。





### 并发和并行

- 并发：同一时刻有多个任务在运行。
- 并行：同一时间有多个任务在运行。





### 线程的创建



线程创建方式争议较多，在Oracle官方文档给出的创建方式为两种，分别是继承Thread类和实现Runnable接口。

&gt; There are two ways to create a new thread of execution. One is to declare a class to be a subclass of `Thread`. This subclass should override the `run` method of class `Thread`. An instance of the subclass can then be allocated and started. 
&gt;
&gt; The other way to create a thread is to declare a class that implements the `Runnable` interface. That class then implements the `run` method. An instance of the class can then be allocated, passed as an argument when creating `Thread`, and started. 

实现Runnable和Callable接口的类只能当作是一个可以在线程中运行的任务，不是真正意义上的线程。

线程对象在构建的时候需要提供线程所需的属性，如线程所属的线程组，线程优先级，是否守护线程等信息，下面贴出`Thread.init()`方法。

- g：线程组，指定该线程在哪个线程组下
- target：指定要执行的任务
- name：线程的名称
- acc：用来初始化私有变量`inheritedAccessControlContext`，`exit`时会将该变量置为`null`，用来做线程的权限控制
- inheritThreadLocals：可继承的`ThreadLocal`

```java
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException(&#34;name cannot be null&#34;);
        }
		//设置线程名称
        this.name = name;
		//设置当前线程为该线程的父线程
        Thread parent = currentThread();
    	
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it&#39;s an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn&#39;t have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
    	//将daemon、priority属性设置位父线程的对应属性
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
    
    	//设置线程上下文类加载器
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
    
    	//将父线程的InheritThreadLocal复制过来
        if (inheritThreadLocals &amp;&amp; parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        //设置指定的堆栈大小
        this.stackSize = stackSize;

        //设置线程ID
        tid = nextThreadID();
    }
```



#### Thread类

```java
static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println(&#34;hello world&#34;);
        }
    }

    public static void main(String[] args) {
        Thread myThread = new MyThread();
        myThread.start();
    }
```



lambda简化代码：

```java
new Thread(() -&gt; {
            System.out.println(&#34;hello world&#34;);
        }).start();
```



##### Runnable接口

`Runnable`是一个函数式接口。提供了`Lambda`方式进行简化。

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```



```java
public static class MyThread implements Runnable {
        @Override
        public void run() {
            System.out.println(&#34;hello world&#34;);
        }
    }

    public static void main(String[] args) {
        new Thread(new MyThread()).start();
    }
```



#### Callable接口

`Callable`与`Runnable`类似，都是函数式接口，不过`Callable`提供的接口有返回值。

```java
@FunctionalInterface
public interface Callable&lt;V&gt; {
    V call() throws Exception;
}
```

`Callable`一般配合`ExecutorService`使用。

```java
&lt;T&gt; Future&lt;T&gt; submit(Callable&lt;T&gt; task);
```



Demo：

```java
public class CallableDemo {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        Future&lt;Integer&gt; result = executor.submit(task);
        // 注意调用get方法会阻塞当前线程，直到得到结果。
        // 所以实际编码中建议使用可以设置超时时间的重载get方法。
        System.out.println(result.get());
    }
}

class Task implements Callable&lt;Integer&gt;{
    @Override
    public Integer call() throws Exception {
        // 模拟计算需要一秒
        Thread.sleep(1000);
        return 2;
    }
}

```



#### Future接口

`Future`就是对具体的`Runnable`或者`Callable`任务执行结果进行取消，查询是否完成，获取结果。

```java
public abstract interface Future&lt;V&gt; {
    //取消任务
    public abstract boolean cancel(boolean paramBoolean);
    //任务是否被取消
    public abstract boolean isCancelled();
    //是否已经完成
    public abstract boolean isDone();
    //用来获取执行结果，产生阻塞，知道任务执行完毕才返回
    public abstract V get() throws InterruptedException, ExecutionException;
    //获取执行结果，指定时间未完成，返回null
    public abstract V get(long paramLong, TimeUnit paramTimeUnit)
            throws InterruptedException, ExecutionException, TimeoutException;
}
```



#### FutureTask

`FutureTask`实现了`Future`接口。`FutureTask`是实现的`RunnableFuture`接口的，而`RunnableFuture`接口同时继承了`Runnable`接口和`Future`接口。

```java
static class Task implements Callable&lt;Integer&gt; {
        @Override
        public Integer call() throws Exception {
            Thread.sleep(1000);
            return 5;
        }
        public static void main(String[] args) throws Exception {
            ExecutorService executor = Executors.newCachedThreadPool();
            FutureTask&lt;Integer&gt; futureTask = new FutureTask&lt;&gt;(new Task());
            executor.submit(futureTask);
            System.out.println(futureTask.get());
        }
    }
```







### 线程组

`ThreadGroup`可以对线程进行批量控制。每个`Thread`必然属于一个`ThreadGroup`。创建新线程是不显式指定，默认将父线程所属线程组设置成自己的线程组。

```java
System.out.println(Thread.currentThread().getThreadGroup().getName());
```



### 线程优先级

现代操作系统基本采用时分的形式调度运行的线程，操作系统会分出一个个时间片，线程会分配到若干个时间片，当线程的时间片用完了就会发生线程调度，并等待着下次分配，线程分配到的时间片多少决定线程使用处理器资源的多少，而线程优先级就是决定线程需要多或少分配一些处理器资源的线程属性。

在`Java`线程中，通过一个priority来控制优先级，优先级范围从1~10，在线程构建的时候，可以通过`setPriority(int)`方法来修改优先级，默认优先级为5，优先级高的线程分配时间片的数量要多于优先级低的线程。

Java只是给操作系统一个优先级的**参考值**，线程最终**在操作系统的优先级**是由操作系统的**线程调度算法**决定的。

可以通过`Thread`中的`setPriority()`方法设置线程的优先级



### 线程的状态

Java线程在运行的生命周期中可能处于6种不同的状态，在给定的时刻，线程只能处于其中的一种状态。



|   状态名称   |                             说明                             |
| :----------: | :----------------------------------------------------------: |
|     NEW      |       初始状态，线程被构建，但是还没有调用start()方法        |
|   RUNNABLE   | 运行状态，Java线程将操作系统中的就绪和运行两种状态统称为运行 |
|   BLOCKED    |                    阻塞状态，表示线程阻塞                    |
|   WAITING    | 等待状态，进入该状态表示当前线程需要等待其他线程做出一些动作（通知或中断） |
| TIME_WAITING | 限期等待状态，该状态不同于WAITING，它是可以在指定的时间自行返回的 |
|  TERMINATED  |                终止状态，表示当前线程执行完毕                |



Java线程状态转换如图所示：

![Java线程状态转换](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/concurrency/IMG_0041.PNG)





### Daemon线程

Daemon线程是一种支持性线程，因为它主要被用作程序中后台调度以及支持性工作。这意味着，当一个JVM种不存在非Daemon线程的时候，JVM将会退出。可以通过`Thread.setDaemon(true)`将线程设置为Daemon线程。

在JVM退出时，Daemon线程中的finally块并不一定会执行。因此，在构建Daemon线程时，不能依靠finally块中的内存来确保执行关闭或清理资源的逻辑。

- 垃圾回收线程是一种守护线程
- Tomcat中的Acceptor和Poller线程都是守护线程。



### interrupt()

中断可以理解为线程的一个标识位属性，它表示一个运行中的线程是否被其他线程进行了中断操作，其他线程通过调用该线程的`interrupt()`方法对其进行中断操作。

线程通过检查自身是否被中断来进行响应。线程通过`isinterrupted()`方法来判断是否被中断，也可以调用静态方法`Thread.interrupted()`对当前线程的中断标识位进行复位。

在许多声明抛出`InterruptedException`的方法中，在抛出`InterruptedException`方法之前，JVM会将该线程的中断标识位清除，然后抛出，此时调用`isinterrupted()`将返回false。



### wait/notify

|    方法名称    |                             描述                             |
| :------------: | :----------------------------------------------------------: |
|    notify()    | 通知一个在对象的waitSet上等待的线程，使其从wait()方法返回，而返回的前提时该线程获得到了对象的锁 |
|  notifyAll()   |                 通知所有等待在该对象上的线程                 |
|     wait()     | 调用该方法的线程进入WAITING状态，**进入waitSet等待**。只有等待另外线程的通知或被中断才会返回，需要注意，调用wait()方法后，会释放对象的锁 |
|   wait(long)   | 超时等待一段时间，这里的参数是毫秒，也就是等待长达n毫秒，如果没有通知就超时返回 |
| wait(long,int) |           对于超时时间更细粒度的控制，可以达到纳秒           |

等待/通知机制，是指一个线程A调用了对象O的`wait()`方法进入等待状态，而另一个线程B调用了对象O的`notify()`或者`notifyAll()`方法，线程A收到通知后从对象O的`wait()`返回，进而执行后续操作。



使用等待/通知机制需要注意以下细节：

1. 使用`wait()`、`notify()`、`notifyAll()`时需要先对调用对象加锁。
2. 调用`wait()`方法后，线程状态由RUNNING变为WAITING，并将当前线程放置到对象的等待队列。
3. `notify()`、`notifyAll()`方法调用后，等待线程依旧不会从`wait()`返回，需要调用`notify()`或`notifyAll()`的线程释放锁之后，等待线程才有机会从`wait()`返回。
4. `notify()`方法将等待队列的一个等待线程从等待队列移到同步队列中，而`notifyAll()`方法则是将等待队列中所有的线程全部移同步队列，被移动的线程状态由WAITING变为BLOCKED。
5. 从`wait()`方法返回的前提是获得了调用对象的锁。

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeUnit;

/**
 * @author ：L1nker4
 * @date ： 创建于  2020/4/4 21:07
 * @description：
 */
public class WaitNotify {

    static boolean flag = true;

    static Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread waitThread = new Thread(new Wait(),&#34;waitThread&#34;);
        waitThread.start();
        TimeUnit.SECONDS.sleep(1);
        Thread notifyThread = new Thread(new Notify(),&#34;notifyThread&#34;);
        notifyThread.start();
    }

    static class Wait implements Runnable{
        @Override
        public void run() {
            //加锁，拥有lock的监视器
            synchronized (lock){
                //当条件不满足时，继续wait，同时释放了lock的锁
                while (flag){
                    try {
                        System.out.println(Thread.currentThread() &#43; &#34;flag is true. wait @&#34; &#43; new SimpleDateFormat(&#34;HH:mm:ss&#34;).format(new Date()));
                        lock.wait();
                    } catch (InterruptedException e){

                    }
                }
                //条件满足时，完成工作
                System.out.println(Thread.currentThread() &#43; &#34;flag is false, running @&#34; &#43; new SimpleDateFormat(&#34;HH:mm:ss&#34;).format(new Date()));
            }
        }
    }


    static class Notify implements Runnable{

        @Override
        public void run() {
            //加锁
            synchronized (lock){
                //获取lock的锁，然后进行通知，通知时不会释放lock的锁
                //直到当前线程释放了lock后，waitThread才能从wait方法返回
                System.out.println(Thread.currentThread() &#43; &#34;hold lock, notify @&#34; &#43; new SimpleDateFormat(&#34;HH:mm:ss&#34;).format(new Date()));
                lock.notifyAll();
                flag = false;
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            //再次加锁
            synchronized (lock){
                System.out.println(Thread.currentThread() &#43; &#34;hold lock again,sleep @&#34; &#43; new SimpleDateFormat(&#34;HH:mm:ss&#34;).format(new Date()));
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

}

```

输出：

```
Thread[waitThread,5,main]flag is true. wait @21:44:08
Thread[notifyThread,5,main]hold lock, notify @21:44:09
Thread[notifyThread,5,main]hold lock again,sleep @21:44:14
Thread[waitThread,5,main]flag is false, running @21:44:19
```

WaitThread首先获取了对象的锁，然后调用对象的`wait()`方法，从而放弃了锁并进入了对象的等待队列`WaitQueue`，进入等待状态，由于WaitThread释放了对象的锁，NotifyThread随后获取了对象的锁，并调用了对象的`notify()`方法，将WaitThread从`WaitQueue`移到`SynchronizedQueue`中，此时WaitThread的状态变为阻塞状态，NotifyThread释放了锁之后，WaitThread再次获取到锁并从`wait()`方法返回继续执行。



本例可以提炼出等待/通知的经典范式，该范式分为两部分，分别针对等待方（消费者）和通知方（生产者）。

消费者遵循如下原则：

1. 获取对象的锁
2. 如果条件不满足，那么调用对象的`wait()`方法，被通知仍要检查条件。
3. 条件满足则执行对应的逻辑。

对应伪代码如下：

```java
synchronized(对象){
    while(条件不满足){
        对象.wait();
    }
    对应的处理逻辑
}
```

生产者遵循如下原则：

1. 获得对象的锁
2. 改变条件
3. 通知所有等待在对象上的线程

对应的伪代码如下：

```java
synchronized(对象){
    改变条件
    对象.notifyAll();
}
```



### Thread.join()

如果线程A执行了`thread.join()`方法，那么当前线程A等待thread线程终止之后才从`thread.join()`返回。





### Thread.sleep()

- 当一个线程执行sleep方法后，调用线程会暂时让出指定时间的执行权，也就是这段时间不参与CPU的调度，**但是该线程持有的锁是不让出的。**指定睡眠时间到了后该函数就会正常返回。
- 调用sleep会从Running状态进入Timed Waiting状态
- 其他线程可以使用`interrupt`方法
- `while(true)`语句中使用`sleep`可以避免CPU空转



### Thread.yield()

- 当一个线程调用yield方法时，当前线程会让出CPU使用权，然后处于就绪状态，线程调度器会获取到一个优先级最高的线程。
- 调用yield会让当前线程从Running进入Runnable状态。
- 具体实现依赖于OS的任务调度器。









---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/java%E7%BA%BF%E7%A8%8B%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/  

