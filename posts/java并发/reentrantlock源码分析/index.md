# ReentrantLock源码分析




# 简介
相对于`synchronized`关键字，`ReentrantLock`具备以下特点：
- 可中断
- 可设置超时时间
- 可设置为公平锁
- 支持多个条件变量



# 源码实现



可重入锁的实现上，主要关注两点：

- 可重入线程的再次获取锁的处理

- 可重入锁的释放机制



## 类的继承关系

`ReentrantLock`实现了`Lock`接口，`Lock`接口定义了锁的通用方法。

```java
public class ReentrantLock implements Lock, java.io.Serializable 
```



## 成员变量

`sync`代表当前`ReentrantLock`使用的获取策略。

```java
private final Sync sync;
private static final long serialVersionUID = 7373984872572414699L;
```





## 构造方法

- 无参构造方法
  - 默认是非公平策略
- 有参构造方法
  - 传入`true`使用公平策略。

```java
/**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```





## 获取锁的策略

`ReentrantLock`内部有三个内部类，其中`Sync`是其它两个类`NonfairSync`和`FairSync`的父类，分别代表着非公平策略和公平策略。



两者之间各有优缺点：

- 公平策略：频繁进行上下文切换，造成较大的资源消耗。
- 非公平策略：存在线程饥饿问题，但是与公平策略相比，少量的上下文切换保证了更大的吞吐量。

### Sync

继承自`AbstractQueuedSynchronizer`，实现了对`state`字段的修改操作。

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
    	//非公平方式尝试获取锁
        final boolean nonfairTryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取AQS的state状态
            int c = getState();
            //为0表示暂无线程占用锁
            if (c == 0) {
                //通过CAS设置state
                if (compareAndSetState(0, acquires)) {
                    //设置成功之后，设置当前线程独占
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //如果当前线程拥有锁，则表示进行重入
            else if (current == getExclusiveOwnerThread()) {
                //添加重入次数
                int nextc = c &#43; acquires;
                if (nextc &lt; 0) // overflow
                    throw new Error(&#34;Maximum lock count exceeded&#34;);
                setState(nextc);
                return true;
            }
            return false;
        }

    	//尝试释放锁资源，全部释放则返回true
        protected final boolean tryRelease(int releases) {
            //c是释放后的资源量
            int c = getState() - releases;
            //如果当前线程不是占有锁的线程，抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            //free是全部释放的标识
            boolean free = false;
            //如果c = 0，说明全部释放资源，可重入环境
            if (c == 0) {
                //设置全部释放标识
                free = true;
                //置空独占线程
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    
		//判断资源是否被当前线程占有
        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don&#39;t need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
		
    	//生成一个条件
        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class
		//返回占用锁的线程
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }
		//如果锁被线程占有，则返回state
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }
		
    	//判断锁是否被线程占有
    	//state不等于0则锁被线程占用
        final boolean isLocked() {
            return getState() != 0;
        }

        //自定义反序列化逻辑
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```



### NonfairSync

`NonfairSync`表示是非公平策略获取锁。

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        //获得锁
        final void lock() {
            //CAS操作设置state为1
            if (compareAndSetState(0, 1))
                //CAS设置成功，则设置独占进程为当前进程
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //锁已经被占用，或者set失败
                //独占方式进行获取
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```



### FairSync

采用公平策略获取锁。

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
		//调用acquire方法，以独占方式获取锁
        final void lock() {
            acquire(1);
        }

        //尝试获取公平锁
        protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取AQS的state
            int c = getState();
            //如果当前没有线程占有锁
            if (c == 0) {
                //判断AQS Queue是否有线程在等待
                //如果没有则直接通过CAS获取锁资源
                if (!hasQueuedPredecessors() &amp;&amp;
                    compareAndSetState(0, acquires)) {
                    //设置当前线程为独占线程
                    setExclusiveOwnerThread(current);
                    //获取成功
                    return true;
                }
            }
            //如果当前线程已经占有锁，则更新可重入信息
            else if (current == getExclusiveOwnerThread()) {
                //更新可重入信息
                int nextc = c &#43; acquires;
                //检查边界
                if (nextc &lt; 0)
                    throw new Error(&#34;Maximum lock count exceeded&#34;);
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```









## 核心方法



### lock

调用`sync.lock()`

```java
public void lock() {
        sync.lock();
}
```



### lockInterruptibly

响应中断的获取锁的方法，调用`AQS.acquireInterruptibly()`完成。

```java
public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
}

public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
}
```



### tryLock

非公平方式尝试获取锁，调用`sync.nonfairTryAcquire(1)`完成。重载方法提供了超时策略，同时响应中断。

```java
public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
}

public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}

public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
}
```



### unlock

调用`sync.release(1)`完成`unlock()`操作。

```java
public void unlock() {
        sync.release(1);
}
```



## 总结

`ReentrantLock`的核心功能主要通过内部类`Sync`完成。而`Sync`继承自`AQS`，通过`AQS`中的`Sync Queue`完成对线程排队的功能。`ReentrantLock`的公平策略和非公平策略通过另外两个内部类`FairSync`、`NonfairSync`实现。

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/reentrantlock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/  

