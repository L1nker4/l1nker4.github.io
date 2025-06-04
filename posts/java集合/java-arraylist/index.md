# ArrayList源码分析




## 简介

`ArrayList`是`List`接口的实现类，其底层通过数组实现。当空间不够会通过内部的扩容机制进行扩容。时间复杂度与数组类似，`ArrayList`是非线程安全的集合，在并发环境下使用会产生错误。

```java
public class ArrayList&lt;E&gt; extends AbstractList&lt;E&gt;
        implements List&lt;E&gt;, RandomAccess, Cloneable, java.io.Serializable
```

`ArrayList`继承了`AbstractList`，实现了`List`接口，提供了添加、删除、修改、遍历等功能

`ArrayList`实现了`RandomAccess`接口，实现该接口表明`ArrayList`支持快速随机访问。

`ArrayList`实现了`Serializable`接口，表明`ArrayList`支持序列化。

`ArrayList`实现了`Cloneable`接口，能被克隆。



## 成员变量

```java
	//默认初始化大小
    private static final int DEFAULT_CAPACITY = 10;
	
	//传入initialCapacity为0时，elementData指向该变量
    private static final Object[] EMPTY_ELEMENTDATA = {};
	
	//不传initialCapacity时，elementData指向该变量
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
	
	//存放数据的数组
    transient Object[] elementData;

    //list大小
    private int size;
```



## 构造方法

`ArrayList`共有三个构造方法。注释如下：

```java
//传递了长度的构造方法
public ArrayList(int initialCapacity) {
    	//边界检查
        if (initialCapacity &gt; 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException(&#34;Illegal Capacity: &#34;&#43;
                                               initialCapacity);
        }
    }

    //无参构造
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    //传递集合
    public ArrayList(Collection&lt;? extends E&gt; c) {
        //获取初始值数组
        elementData = c.toArray();
        //如果传入的为非空集合
        if ((size = elementData.length) != 0) {
            // c.toArray可能返回的不是Object[]类型
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            //传递空集合，将elementData指向空数组
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```



## API



### get

获取对应下标的元素，时间复杂度`O(1)`

```java
public E get(int index) {
        rangeCheck(index);
        return elementData(index);
}


E elementData(int index) {
        return (E) elementData[index];
}
```



### rangeCheck

检查传入下标是否合法。

```java
private void rangeCheck(int index) {
        if (index &gt;= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```



### set

设置对应下标的元素，返回原来的元素。

```java
public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```



### add

add有两种方式，一种是尾插，一种是指定位置插入。

插入中，首先需要检查容量是否足够，不够会进行扩容，扩容机制会在后续介绍。

```java
public boolean add(E e) {
    	//判断是否需要扩容
        ensureCapacityInternal(size &#43; 1);  // Increments modCount!!
        elementData[size&#43;&#43;] = e;
        return true;
}


public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size &#43; 1);  // Increments modCount!!
    	//从index开始到末尾，逐个向后移动1位，相当于index处空出一个位置。
        System.arraycopy(elementData, index, elementData, index &#43; 1,
                         size - index);
        elementData[index] = element;
        size&#43;&#43;;
    }
```



### remove

remove有两种删除方式：一种是根据下标删除，一种是根据对象删除。

```java
public E remove(int index) {
        rangeCheck(index);

        modCount&#43;&#43;;
        E oldValue = elementData(index);
		//找到index后一位
        int numMoved = size - index - 1;
        if (numMoved &gt; 0)
            //将numMoved开始到结尾向前移动一位
            System.arraycopy(elementData, index&#43;1, elementData, index,
                             numMoved);
    	//置空，让其被GC
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
}

public boolean remove(Object o) {
    	//检查
        if (o == null) {
            for (int index = 0; index &lt; size; index&#43;&#43;)
                //判断空元素
                if (elementData[index] == null) {
                    //fastRemove不检查边界值，不返回删除元素值
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index &lt; size; index&#43;&#43;)
                //调用equals检查对象是否符合要求
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
}
```



### fastRemove

不检查边界值，并且不返回删除元素的方式进行删除指定下标的数据。

```java
private void fastRemove(int index) {
        modCount&#43;&#43;;
        int numMoved = size - index - 1;
        if (numMoved &gt; 0)
            System.arraycopy(elementData, index&#43;1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
}
```



### clear

清空列表。

```java
public void clear() {
        modCount&#43;&#43;;

        // 将每个元素置空，方便GC回收
        for (int i = 0; i &lt; size; i&#43;&#43;)
            elementData[i] = null;
		//长度置为0
        size = 0;
}
```



### addAll

将集合添加进列表中。

```java
public boolean addAll(Collection&lt;? extends E&gt; c) {
    	//将集合转为Object数组
        Object[] a = c.toArray();
        int numNew = a.length;
    	//确保容量足够
        ensureCapacityInternal(size &#43; numNew);  // Increments modCount
    	//将数组copy到elementData中
        System.arraycopy(a, 0, elementData, size, numNew);
    	//更新长度
        size &#43;= numNew;
        return numNew != 0;
    }
```



### indexOf

返回元素在列表中第一个匹配的下标。不存在返回-1。`lastIndexOf`同理，找到最后匹配的下标。实现原理为`for`从前往后扫描。

```java
public int indexOf(Object o) {
    	//如果传入元素为空
        if (o == null) {
            //找到列表中第一个为空的下标，返回
            for (int i = 0; i &lt; size; i&#43;&#43;)
                if (elementData[i]==null)
                    return i;
        } else {
            //equals找到第一个匹配对象，并返回
            for (int i = 0; i &lt; size; i&#43;&#43;)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```



### removeRange

删除指定区间的元素。

```java
protected void removeRange(int fromIndex, int toIndex) {
        modCount&#43;&#43;;
    	//将指定区间的元素覆盖掉
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        //将指定区间的元素置空，方便GC
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i &lt; size; i&#43;&#43;) {
            elementData[i] = null;
        }
        size = newSize;
    }
```



### retainAll

retainAll返回该集合与传入集合的交集，并且将非交集元素删除。通过调用`batchRemove`进行元素删除。

```java
public boolean retainAll(Collection&lt;?&gt; c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }

//删除非交集元素
private boolean batchRemove(Collection&lt;?&gt; c, boolean complement) {
    	//获取到当前List的数组
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            //for找出相同的，并将相同的逐个放在elementData
            for (; r &lt; size; r&#43;&#43;)
                if (c.contains(elementData[r]) == complement)
                    elementData[w&#43;&#43;] = elementData[r];
        } finally {
            //如果 c.contains()抛出异常
            if (r != size) {
                //复制剩余的元素
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                //w为当前List的length
                w &#43;= size - r;
            }
            //如果交集长度与原来长度不匹配
            if (w != size) {
                // 删除多余的元素
                for (int i = w; i &lt; size; i&#43;&#43;)
                    elementData[i] = null;
                modCount &#43;= size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```



### trimToSize

手动缩容，将list的elementData没使用的空间删除。

```java
public void trimToSize() {
        modCount&#43;&#43;;
        if (size &lt; elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```



## 扩容机制

`ensureCapacityInternal`提供内部调用，`ensureCapacity`可提供外部调用。

- 扩容时机：`size &#43; 1 - elementData.length &gt; 0`，即list实际大小超出elementData长度时，进行扩容
- 扩容倍数：原容量的1.5倍（向下取整）


```java
private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

private void ensureExplicitCapacity(int minCapacity) {
        modCount&#43;&#43;;

        // 边界检查
        if (minCapacity - elementData.length &gt; 0)
            grow(minCapacity);
    }

private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It&#39;s already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity &gt; minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }


private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;


private void grow(int minCapacity) {
        // 原容量
        int oldCapacity = elementData.length;
    	//新容量为原容量的1.5倍
        int newCapacity = oldCapacity &#43; (oldCapacity &gt;&gt; 1);
    	//判断新容量是否大于需要的最小容量
        if (newCapacity - minCapacity &lt; 0)
            newCapacity = minCapacity;
    	//比最大容量还大
        if (newCapacity - MAX_ARRAY_SIZE &gt; 0)
            newCapacity = hugeCapacity(minCapacity);
        //copy
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

private static int hugeCapacity(int minCapacity) {
        if (minCapacity &lt; 0) // overflow
            throw new OutOfMemoryError();
    	//若minCapacity &gt; MAX_ARRAY_SIZE,将Integer.MAX_VALUE作为新数组大小
    	//否则MAX_ARRAY_SIZE作为新数组大小
        return (minCapacity &gt; MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```





## modCount

在对List的结构进行修改时，都会进行`modCount&#43;&#43;`，该变量记录集合的修改次数。其中包括：

- trimToSize
- ensureExplicitCapacity
- remove
- clear
- removeIf
- replaceAll
- sort



在使用迭代器遍历集合的时候同时修改结构时，`modCount`值会改变，而迭代器中`checkForComodification`方法是在迭代中检查集合是否发生修改。其原理是比较创建迭代器的时候的`modCount`与当前modCount是否相同。

```java
final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
```



## 迭代器

`next`方法会调用`checkForComodification`方法来判断`List`迭代过程中是否发生结构上的修改，原理已在上文阐述。

```java
/**
     * An optimized version of AbstractList.Itr
     */
private class Itr implements Iterator&lt;E&gt; {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        Itr() {}

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings(&#34;unchecked&#34;)
    	//通过i = cursor自增，逐个获取元素
        public E next() {
            //判断List是否在迭代器创建后发生过修改
            checkForComodification();
            int i = cursor;
            if (i &gt;= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i &gt;= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i &#43; 1;
            return (E) elementData[lastRet = i];
        }

       ......
@Override
        @SuppressWarnings(&#34;unchecked&#34;)
        public void forEachRemaining(Consumer&lt;? super E&gt; consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i &gt;= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i &gt;= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size &amp;&amp; modCount == expectedModCount) {
                consumer.accept((E) elementData[i&#43;&#43;]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }
    
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```



#### 遍历时删除

如果在遍历时删除，foreach在编译成字节码时会转成迭代器遍历的方式。运行时会在上述`checkForComodification`方法中抛出`ConcurrentModificationException`。

```java
public static void main(String[] args) {
        ArrayList&lt;Integer&gt; list = new ArrayList&lt;&gt;();
        list.add(2);
        list.add(1);
        list.add(3);
        list.add(1);
        System.out.println(list);
        for (Integer item : list) {
            if (item.equals(1)){
                list.remove(item);
            }
        }
        System.out.println(list);
}
```

上述机制称为`fail-fast`机制，这种机制在Java集合类中十分常见，简而言之就是操作前先考虑异常情况，如果发生异常，立即停止操作。



根据阿里的Java规范，正确的方式如下：

&gt; 【强制】不要在 foreach 循环里进行元素的 remove/add 操作。remove 元素请使用 Iterator
&gt; 方式，如果并发操作，需要对 Iterator 对象加锁。



```java
Iterator&lt;Integer&gt; it = list.iterator();
        while (it.hasNext())
        {
            Integer s = it.next();
            if (s.equals(1))
            {
                it.remove();
            }
        }
        System.out.println(list);
```





## 序列化

`ArrayList`的`elementData `添加了`transient`关键字，所以在序列化的时候`elementData `并不是序列化的一部分。而`ArrayList`通过内部更为细致的方法进行了序列化控制。内部序列化方法只保存非空元素，从而达到节约空间的目的。

```java
 private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i&lt;size; i&#43;&#43;) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * Reconstitute the &lt;tt&gt;ArrayList&lt;/tt&gt; instance from a stream (that is,
     * deserialize it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size &gt; 0) {
            // be like clone(), allocate array based upon size not capacity
            int capacity = calculateCapacity(elementData, size);
            SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i&lt;size; i&#43;&#43;) {
                a[i] = s.readObject();
            }
        }
    }
```



## 子列表



`subList`返回`List`指定区间的数据，可以看到`subList`中的操作会影响到父列表。

其次需要注意的是，`subList`的返回结果不可强转成`ArrayList`，否则会抛出`ClassCastException`，因为`subList`返回的是`ArrayList`的内部类`SubList`。

```java
public List&lt;E&gt; subList(int fromIndex, int toIndex) {
            subListRangeCheck(fromIndex, toIndex, size);
            return new SubList(this, offset, fromIndex, toIndex);
        }

private class SubList extends AbstractList&lt;E&gt; implements RandomAccess {
        private final AbstractList&lt;E&gt; parent;
        private final int parentOffset;
        private final int offset;
        int size;

        SubList(AbstractList&lt;E&gt; parent,
                int offset, int fromIndex, int toIndex) {
            this.parent = parent;
            this.parentOffset = fromIndex;
            this.offset = offset &#43; fromIndex;
            this.size = toIndex - fromIndex;
            this.modCount = ArrayList.this.modCount;
        }

        public E set(int index, E e) {
            rangeCheck(index);
            checkForComodification();
            E oldValue = ArrayList.this.elementData(offset &#43; index);
            ArrayList.this.elementData[offset &#43; index] = e;
            return oldValue;
        }

        public E get(int index) {
            rangeCheck(index);
            checkForComodification();
            return ArrayList.this.elementData(offset &#43; index);
        }
    
}

```



## Arrays.asList

该方法通常用于将一个数组转换成一个List集合，但是需要注意不能使用其修改集合的相关方法，add/remove/clear会抛出`UnsupportedOperationException`，因为该方法返回的`ArrayList`是`Arrays`的一个内部类。其内部没有实现集合的修改方法。



其次传递的数组必须是对象数组，而不能是基本类型，如果传入基本类型数组，该方法得到的就是数组对象本身，而不是元素。

```java
@SafeVarargs
    @SuppressWarnings(&#34;varargs&#34;)
    public static &lt;T&gt; List&lt;T&gt; asList(T... a) {
        return new ArrayList&lt;&gt;(a);
    }
```



演示：

```java
public static void main(String[] args) {
        List&lt;String&gt; list = Arrays.asList(&#34;hello&#34;, &#34;world&#34;);
        list.add(&#34;yes&#34;);

        int[] arr = {1,2,3,4,5};
        List&lt;int[]&gt; list1 = Arrays.asList(arr);
        System.out.println(list1.get(2));
    }
```





## 总结

本文对`ArrayList`的成员变量，构造方法，API，扩容机制与其它细节做了简单介绍。并对日常开发中会使用到的一些细节做了简要描述，可以减少在日常开发过程中犯错误的几率。

---

> Author:   
> URL: http://localhost:1313/posts/java%E9%9B%86%E5%90%88/java-arraylist/  

