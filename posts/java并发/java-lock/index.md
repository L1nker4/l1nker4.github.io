# 解析Java中的锁




## 简介

锁是用来控制多个线程访问共享资源的方式，在Lock接口出现之前，Java是靠synchronized关键字实现锁功能的。而Java 1.5之后，并发包中新增了Lock接口与其实现类用来实现锁功能，只是需要手动获取释放锁，虽然它缺少了同步关键字隐式获取释放的便捷性，但却拥有了可操作性，可中断的获取锁以及超时获取锁等功能。



## 分类

Java中会按照是否有某一特性来定义锁，下图通过各种特性对锁进行分类：

![Java中的锁](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/concurrency/lock/Java%E7%9A%84%E9%94%81.png)



### 悲观锁 / 乐观锁

这两种锁不是具体类型的锁，体现了看待线程同步的角度，再Java和数据库中都有此概念对应的实际应用。

对于同一个数据的并发操作，悲观锁认为自己在使用数据的时候，一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改，在Java中，synchronized关键字和Lock的实现类都是悲观锁。

而乐观锁认为自己在使用数据的时候，不会有其他线程修改数据，所以不会添加锁，只是在更新数据的时候，去判断之前有没有别的线程更新了数据，如果没有被更新，当前线程将自己修改的数据成功写入。如果数据已被其他线程更新。则根据不同的实现方式执行不同的操作（报错或自动重试）。

乐观锁在Java中是通过无锁编程来实现，最常采用是CAS算法，Java原子类中的递增就是通过CAS自旋来实现的。

悲观锁适合写操作多的场景，先加锁可以保证数据准确性。

乐观锁适合读操作多的场景，不加锁能够提高性能。





### 自旋锁 / 适应性自旋锁

在Java中，自旋锁是指尝试获取锁的线程不会立即阻塞，而实采用循环的方式去获取锁，这样做的好处是减少线程上下文切换的消耗。

但是自旋锁本身是有缺点的，它不能代替阻塞，自旋虽然避免了上下文切换的开销，但它要占用处理器时间，如果锁被占用的时间很短，自旋等待的效果很好，但是如果锁占用时间过长，自旋只会白白浪费处理器资源。所以自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，通过**-XX:PreBlockSpin**修改）没有成功获得锁，就挂起线程，停止自旋。

自旋锁的实现原理是CAS算法。自旋锁在JDK 1.4.2引入，使用**-XX:UseSpinning**开启，JDK 6开始默认开启，并且引入了自适应的自旋锁。

自适应意味着自旋的时间不再固定，而实由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么JVM会认为这次自选也是很有可能再次成功，进而它将自旋等待持续更长的时间。如果某个锁自旋很少成功获得，那么就会直接省略掉自旋过程，直接阻塞线程。

在自旋锁中，有三种常见的锁形式：TicketLock、CLHlock、MCSlock





### 无锁 / 偏向锁 / 轻量级锁 / 重量级锁

这四种指锁的状态，并且是针对`Synchronized`关键字，是通过`Mark Word`中的字段表明的。



#### 无锁

无锁没有对资源进行锁定，所有线程都能访问并修改同一个资源，但同时只有一个线程能修改成功。

无锁的特点是修改操作在循环内进行，线程会不断尝试修改共享资源。如果没有冲突就修改成功并退出，否则就会继续循环尝试。CAS原理就是无锁的实现。



#### 偏向锁

偏向锁是指一段同步代码一直被一个线程所访问，那么该线程就会自动获得锁，降低获得锁的代价。

当一个线程通过同步代码块获得锁的时候，会在`Mark Word`中存储锁偏向的线程ID。在线程进入或退出同步代码块时不再通过CAS操作来加锁解锁，而是检查`Mark Word`中是否存储着指向当前线程的偏向锁，引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁的执行，因为轻量级锁较偏向锁消耗性能。

偏向锁只有遇到其他线程竞争偏向锁时，持有偏向锁的线程才会释放偏向锁，线程不会主动释放偏向锁。

偏向锁在JDK 6以后是默认启用的，可以通过`-XX:UseBiasedLocking=false`关闭，关闭之后，程序默认进入轻量级锁状态。



#### 轻量级锁

轻量级锁是指当锁是偏向锁的时候，被另一个线程访问，偏向锁就会升级为轻量级锁，其他线程通过自旋的方式尝试获取锁，不会阻塞。从而提高性能。



#### 重量级锁

若当前只有一个等待线程，则该线程通过自旋进行等待，但是当自旋超过一定次数，或是一个线程在持有锁，一个在自旋，又有第三个线程访问时，轻量级锁升级为重量级锁。



综上，偏向锁通过对比`Mark Word`解决加锁问题，避免执行CAS操作，而轻量级锁通过CAS操作和自旋来解决加锁问题，避免线程阻塞和唤醒影响性能。重量级锁将除了拥有锁的线程以外所有线程都阻塞。



### 公平锁 / 非公平锁

公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列进行排序，队列中第一个线程才能获得锁。

公平锁的优点时等待的线程不会饿死，缺点是整体吞吐效率相对非公平锁较低，等待队列中除第一个线程以外所有线程都阻塞，CPU唤醒阻塞线程的开销较非公平锁大。

非公平锁是多个线程加锁时直接尝试获得锁，获得不到才会进入等待队列中等待。如果此时锁刚好可用，那么线程可以无需阻塞直接获取到锁。非公平锁的优点是可以减少唤醒线程的开销，整体吞吐效率高，因为线程有几率不阻塞直接获得锁，缺点是处于等待队列的线程可能会饿死，或者等待很久才能获得锁。





### 可重入锁

可重入锁又称为递归锁，是指同一个线程在外层方法获取锁的时候，在进入内层方法会自当获得锁（前提是锁对象是同一个对象），不会因为之前获取过还没释放而阻塞，Java中`ReentrantLock`和`Synchronized`都是可重入锁，可重入锁的一个优点就是可一定程度避免死锁。

下面是一个可重入锁的一个案例。

```java
synchronized void setA() throws Exception{
	Thread.sleep(1000);
	setB();
}

synchronized void setB() throws Exception{
	Thread.sleep(1000);
}
```



#### 独享锁 / 共享锁

独享锁也叫排他锁，是指该锁一次只能被一个线程所持有，如果线程T对数据A加上独享锁之后，则其他线程不再对A加任何类型的锁，获得独享锁的数据即能读数据又能修改数据。

共享锁是指该锁可被多个线程所持有，如果线程T对数据A加上共享锁之后，则其他线程只能对A加共享锁，而不能加独享锁，获得到共享锁的线程只能读数据，而不能修改数据。

独享锁和共享锁通过AQS实现。通过实现不同的方法来实现独享或共享。在Java中，`ReentrantLock`、`Synchronized`和`ReadWriteLock`都是独享锁。





## 参考

《Java并发编程的艺术》

[Java中的锁分类](https://www.cnblogs.com/qifengshi/p/6831055.html)

[不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)



---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/java-lock/  

