# ThreadLocal解析


# 简介

&gt; This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its `get` or `set` method) has its own, independently initialized copy of the variable. `ThreadLocal` instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

简而言之，`ThreadLocal`可以创建一个只能被当前线程访问或修改的变量。





# 分析



&gt; Q：如何实现线程隔离？



使用`Thread`对象中的`ThreadLocalMap`进行数据存储。也就是`ThreadLocal`将数据存储到当前的线程对象中，通过`Thread.currentThread()`来获取线程，再通过`getMap(t)`来获取`ThreadLocalMap`。具体内容通过阅读源码来逐步分析。



## get

返回当前线程存储的`ThreadLocal`值，如果不存在，会进行初始化并返回。通过`map.getEntry(this)`。

```java
/**
 * Returns the value in the current thread&#39;s copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread&#39;s value of this thread-local
 */
public T get() {
    //获取当前对象
    Thread t = Thread.currentThread();
    //通过getMap获取ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //获取entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings(&#34;unchecked&#34;)
            T result = (T)e.value;
            return result;
        }
    }
    //不存在则进行初始化
    return setInitialValue();
}
```



## getMap

返回指定线程的`ThreadLocalMap`。

```java
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
}
```



`Thread`中关于`ThreadLocalMap`的部分：

```java
public class Thread implements Runnable {
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

}
```



## setInitialValue

get方法获取不到值时，通过该方法设置初始值。

```java
/**
 * Variant of set() to establish initialValue. Used instead
 * of set() in case user has overridden the set() method.
 *
 * @return the initial value
 */
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```



## createMap

`ThreadLocalMap`不存在时，通过该方法创建，这里有一个疑惑：

&gt;  Q：为什么在`get`和`setInitialValue`进行两次为空检查才进行`createMap`？

```java
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```



## set

通过调用`map.set(this,value)`实现。map中的key是当前`ThreadLocal`对象。

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```



## ThreadLocalMap



内部类和成员变量代码如下：

```java
static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as &#34;stale entries&#34; in the code that follows.
         */
        static class Entry extends WeakReference&lt;ThreadLocal&lt;?&gt;&gt; {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal&lt;?&gt; k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0
}
```



### set

流程：

- 计算下标，得到下标i
- 进行遍历，如果i对应的key和传入相等，直接返回。如果对象已被回收，调用`replaceStaleEntry()`并返回。
- 如果i对应的不相等，则从i开始从table数组完后找。
- 能找到则返回，找不到新建entry进行存储。
- 检查是否需要`rehash()`

```java
/**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal&lt;?&gt; key, Object value) {

            // We don&#39;t use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.
			
            Entry[] tab = table;
            int len = tab.length;
            //获取hash后的下标
            int i = key.threadLocalHashCode &amp; (len-1);
			//遍历
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal&lt;?&gt; k = e.get();
				//如果对应下标的ThreadLocal与当前的相等，直接更新
                if (k == key) {
                    e.value = value;
                    return;
                }
				//如果ThreadLocal对象已被回收，调用replaceStaleEntry
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
			//如果遍历一圈还是找不到，新建entry进行存储
            tab[i] = new Entry(key, value);
            int sz = &#43;&#43;size;
            //检查是否需要rehash
            if (!cleanSomeSlots(i, sz) &amp;&amp; sz &gt;= threshold)
                rehash();
        }
```



下标计算方式：

通过`AtomicInteger &#43; 0x61c88647 &amp; (len-1)`来获取下标。

```java
int i = key.threadLocalHashCode &amp; (len-1);

private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```



### getEntry

流程：

- 计算下标，得到下标i
- 判断对应下标的entry是否存在，key是否相等，符合要求则直接返回。
- 不符合则调用`getEntryAfterMiss()`

```java
/**
 * Get the entry associated with key.  This method
 * itself handles only the fast path: a direct hit of existing
 * key. It otherwise relays to getEntryAfterMiss.  This is
 * designed to maximize performance for direct hits, in part
 * by making this method readily inlinable.
 *
 * @param  key the thread local object
 * @return the entry associated with key, or null if no such
 */
private Entry getEntry(ThreadLocal&lt;?&gt; key) {
    //计算下标
    int i = key.threadLocalHashCode &amp; (table.length - 1);
    Entry e = table[i];
    if (e != null &amp;&amp; e.get() == key)
        return e;
    else
        //entry不存在或key不相等
        return getEntryAfterMiss(key, i, e);
}
```



#### getEntryAfterMiss

也就是对table数组进行遍历查找，找一圈还没有则返回`null`。

```java
private Entry getEntryAfterMiss(ThreadLocal&lt;?&gt; key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal&lt;?&gt; k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```





### remove

流程：

- 计算下标
- 判断key是否相等，符合要求则调用`e.clear()`来清除key，并调用`expungeStaleEntry`清除value。

```java
/**
         * Remove the entry for key.
         */
        private void remove(ThreadLocal&lt;?&gt; key) {
            Entry[] tab = table;
            int len = tab.length;
            //计算下标
            int i = key.threadLocalHashCode &amp; (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                //如果key相等
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```



# 内存泄露问题



## 问题分析

从上文可以看到，**Entry**继承自**WeakReference**，弱引用指向的对象会在下一次GC时被回收。

```java
Entry(ThreadLocal&lt;?&gt; k, Object v) {
                super(k);
                value = v;
}
```

​		在初始化时，将弱引用指向`ThreadLocal`实例。如果外部没有强引用指向ThreadLocal的话，那么Thread Local实例就没有一条引用链路可达，则会被回收。此时entry的key未null，但是有value，但是不能通过key找到value，这样就会存在**内存泄漏问题**。但是如果线程结束，以上内存都会被回收，也就不存在上述问题。

​		如果使用线程池去维护线程，线程池并不会主动销毁内部的线程，总会存在着强引用，那么还是会存在问题。



总结问题出现的原因：

1. 线程一直运行，不终止（线程池）
2. null-value：某个弱引用key被回收



## 如何解决

上述的`getEntryAfterMiss()`如果判断到了`key == null`则会调用`expungeStaleEntry()`将value删除。

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal&lt;?&gt; k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode &amp; (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```



该方法在`set`、`get`、`remove`中都会调用。因此还有一个问题：如果创建了`ThreadLocal`但不调用以上方法，还是会存在问题。

所以最能解决办法的就是用完`ThreadLocal`之后显式执行`ThreadLocal.remove()`。







## 为什么key不设置为强引用

既然弱引用会存在内存泄露问题，为什么不使用强引用呢？



如果将key设置为强引用，当`ThreadLocal`实例释放后，但是ThreadLocal强引用指向threadLocalMap，threadLocalMap.Entry又强引用指向ThreadLocal，这样会导致ThreadLocal无法回收。也就会出现更为严重的问题。



# 参考

Java SE 8 Docs API

https://www.jianshu.com/p/dde92ec37bd1

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%B9%B6%E5%8F%91/java-threadlocal/  

