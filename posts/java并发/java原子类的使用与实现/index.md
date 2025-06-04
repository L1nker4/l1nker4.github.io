# Java原子类的使用与实现




## 简介

通常情况下，在Java中，`&#43;&#43;i`这类自增/自减运算符在并发环境中不能保证并发安全。需要通过加锁才能解决并发环境下的原子性问题。Atomic原子类通过CAS方式来解决线程安全问题，CAS是一种无锁算法（乐观锁），乐观锁多用于“读多写少“的环境，避免频繁加锁影响性能；而悲观锁多用于”写多读少“的环境，避免频繁失败和重试影响性能。



Atomic原子类分为以下几类：

- 基本类型：AtomicInteger，AtomicLong，AtomicBoolean
- 数组类型：AtomicIntegerArray，AtomicLongArray，AtomicReferenceArray
- 引用类型：AtomicReference，AtomicStampedRerence，AtomicMarkableReference
- 更新字段类：AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater
- Java8 新增类：DoubleAccumulator，DoubleAdder，LongAccumulator，LongAdder



## 使用



### 原子基本类型

使用原子的方式更新基本类型。

- AtomicBoolean: 原子布尔类型。
- AtomicInteger: 原子整型。
- AtomicLong: 原子长整型。

以下以`AtomInteger`举例：

```java
//以原子方式将给定值与当前值相加，线程安全的i = i &#43; delta
int addAndGet(int delta);
//如果当前值== except，则以原子方式将当前值设置为update，成功返回true
boolean compareAndSet(int expect, int update);
//以原子方式将当前值减1，相当于线程安全的i--
int decrementAndGet();
//以原子方式将当前值加1，相当于线程安全的i&#43;&#43;
int incrementAndGet()


private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField(&#34;value&#34;));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

/**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) &#43; 1;
    }

    /**
     * Atomically decrements by one the current value.
     *
     * @return the updated value
     */
    public final int decrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }
```

从上面代码可以看出，AtomicInteger底层使用`volatile`关键字和CAS来保证线程安全。其中：
- volatile保证线程的可见性，让每次读取的变量都是最新值
- CAS保证原子性



### 原子数组

以原子方式更新数组中的某个元素。
- AtomicIntegerArray: 原子更新整型数组里的元素。
- AtomicLongArray: 原子更新长整型数组里的元素。
- AtomicReferenceArray: 原子更新引用类型数组里的元素。

``` java
public static void main(String[] args) throws InterruptedException {
        AtomicIntegerArray array = new AtomicIntegerArray(new int[] { 0, 0 });
        System.out.println(array);
        System.out.println(array.getAndAdd(1, 2));
        System.out.println(array);
    }
```



### 原子引用类型

- AtomicReference：原子更新引用类型。
- AtomicStampedReference： 原子更新引用类型,内部使用Pair来存储元素值及其版本号。
- AtomicMarkableReferce：原子更新带有标记位的引用类型，标记是否被改过。

``` java
// 创建两个Person对象，它们的id分别是101和102。
Person p1 = new Person(101);
Person p2 = new Person(102);
// 新建AtomicReference对象，初始化它的值为p1对象
AtomicReference ar = new AtomicReference(p1);
// 通过CAS设置ar。如果ar的值为p1的话，则将其设置为p2。
ar.compareAndSet(p1, p2);
Person p3 = (Person)ar.get();
System.out.println(&#34;p3 is &#34;&#43;p3);
System.out.println(&#34;p3.equals(p1)=&#34;&#43;p3.equals(p1));


//output

p3 is id:102
p3.equals(p1)=false
```



### 原子字段类

- AtomicIntegerFieldUpdater：原子更新Integer的字段的更新器。 
- AtomicLongFieldUpdater：原子更新Long字段的更新器。 
- AtomicStampedFieldUpdater： 原子更新带有版本号的引用类型。
- AtomicReferenceFieldUpdater： 原子更新引用的更新器。

这四个类通过反射更新字段的值，使用字段类如下：
1. 使用静态方法`newUpdater()`创建一个更新器，并需要设置想要更新的类和属性。
2. 更新类的字段必须使用public volatile修饰。

```java
public class AtomicIntegerFieldUpdaterTest {


     private static Class&lt;Person&gt; cls;
     
     public static void main(String[] args) {
        AtomicIntegerFieldUpdater&lt;Person&gt; mAtoLong = AtomicIntegerFieldUpdater.newUpdater(Person.class, &#34;id&#34;);
        Person person = new Person(1000);
        mAtoLong.compareAndSet(person, 1000, 1001);
        System.out.println(&#34;id=&#34;&#43;person.getId());
     }
}

class Person {
    volatile int id;
    public Person(int id) {
        this.id = id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public int getId() {
        return id;
    }
}
```

使用该类有如下约束：
- 字段必须由volatile修饰
- 字段的访问修饰符能让调用方访问到。
- 只能是实例变量，不能加static
- 对于AtomicIntegerFieldUpdater和AtomicLongFieldUpdater只能修改int/long类型的字段，不能修改其包装类型(Integer/Long)。如果要修改包装类型就需要使用AtomicReferenceFieldUpdater。 



## CAS理论

CAS（Compare-And-Swap）：是一条CPU的原子指令，其作用是让CPU先进行比较两个值是否相等，然后原子地更新某个位置的值。一个当前内存值V、旧的预期值A、即将更新的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则返回false。CAS的基本思路就是循环进行CAS操作直到成功为止。

CAS在X86架构下汇编指令是`lock cmpxchg`。需要和volatile配合使用，保证拿到的变量是最新值。





虽然乐观锁通常情况下比悲观锁性能更优，但是还存在以下一些问题：

### ABA问题

CAS自旋需要在操作值得时候检查是否发生变化，但是如果一个值是A，变成B，然后又变成A，CAS检查会发现没有变化。**AtomicStampedReference**来解决ABA问题：这个类的`compareAndSet`方法作用是首先检查当前引用是否等于预期引用，并且当前`stamp`是否等于预期`stamp`，这里的`stamp`类似于版本号功能。如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。



### 循环时间长开销大

CAS自旋如果长时间不成功，会给CPU带来较大得执行开销。




### 只能保证一个共享变量的原子操作

对多个共享变量操作时，CAS无法保证多个操作的原子性，但是可以使用锁来解决。
当然可以将多个共享变量合并成一个共享变量来操作，比如`i = 2;j = a`，合并为`ij = 2a`，然后CAS操作`ij`，从Java 1.5开始，JDK提供了`AtomicReference`类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。





## Unsafe-CAS解析

Java中原子类中CAS操作通过调用`Unsafe`去实现，`Unsafe`是位于`sun.misc`下的一个类，提供的功能主要如下：
- 数组相关
    - 返回数组元素内存大小
    - 返回数组首元素偏移大小
- 内存屏障
    - 禁止load, store重排序
- 系统相关
    - 返回内存页的大小
    - 返回系统指针大小
- 线程调度
    - 线程挂起, 恢复
    - 获取，释放锁
- 内存操作
    - 分配、拷贝、扩充、释放堆外内存
- CAS
- Class
    - 动态创建类
    - 获取field的内存地址偏移量
    - 检测、确保类初始化
- 对象操作
    - 获取对象成员属性的偏移量
    - 非常规对象实例化
    - 存储、获取指定偏移地址的变量值。



下文对`Unsafe`的CAS做简要分析。

```java
public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }

    public final long getAndSetLong(Object var1, long var2, long var4) {
        long var6;
        do {
            var6 = this.getLongVolatile(var1, var2);
        } while(!this.compareAndSwapLong(var1, var2, var6, var4));

        return var6;
    }
```



`Unsafe`调用`compareAndSwapInt`进行CAS操作，使用while操作进行自旋。这是一个`native`方法，实现位于[unsafe.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/tip/src/share/vm/prims/unsafe.cpp)，`cmpxchg`在不同平台上有不同的实现。

```C&#43;&#43;
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper(&#34;Unsafe_CompareAndSwapInt&#34;);
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```



---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/java%E5%8E%9F%E5%AD%90%E7%B1%BB%E7%9A%84%E4%BD%BF%E7%94%A8%E4%B8%8E%E5%AE%9E%E7%8E%B0/  

