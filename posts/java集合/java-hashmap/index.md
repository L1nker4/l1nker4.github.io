# HashMap源码分析




## 概述

​	Java中对于Map数据结构，提供了`java.util.Map`接口，该接口下主要有四个常见的实现类，分别是：

- `HashMap`：根据`key.hashCode`计算存储位置，能在$O(1)$完成查询，但是遍历顺序是不确定的，`HashMap`最多存储一个键为null，允许多条entry的值为null。`HashMap`非线程安全。
- `HashTable`：线程安全的Map，常用方法全部通过`synchronized`保证线程安全，可以使用`ConcurrentHashMap`达到目的，此类不建议使用。
- `LinkedHashMap`：继承自`HashMap`，内部通过双向链表将所有entry连接起来，保证了迭代顺序和插入顺序相同。
- `TreeMap`：实现了SortedMap接口，能够将保存的记录按照键排序。内部通过红黑树进行存储。



`HashMap`在`JDK 7`中采用了数组&#43;链表的数据结构，在`JDK 8`后，底层数据结构转变为：数组&#43;链表&#43;红黑树，也就是当出现Hash冲突时，当链表长度大于阈值(或者红黑树的边界值，默认为8)并且当前数组的长度大于64，链表会转为红黑树存储节点提高搜索效率。

```java
public class HashMap&lt;K,V&gt; extends AbstractMap&lt;K,V&gt; implements Map&lt;K,V&gt;, Cloneable, Serializable
```

`HashMap` 继承了 `AbstractMap` ，该类提供了Map接口的抽象实现，并提供了一些方法的基本实现。实现了 `Map`、`Cloneable`和 `Serializable`接口。



## 成员变量

- loadFactor：该变量控制table数组存放数据的疏密程度，越趋向1时，数组中存放的数据越多越密。链表的长度会增加，因此会导致查找效率变低。该值越小，则数组中存放的数据越少，越稀疏，则会导致空间利用率下降。默认值0.75是较好的默认值，可以最大程度减少rehash的次数，避免过多的性能消耗。
- threshold：当前 `HashMap`所能容纳键值对数量的最大值，超过这个值，则需扩容。**threshold = capacity \* loadFactor**
  - 默认容量为16，默认负载因子为0.75，当size达到16 * 0.75 = 12时，需要进行扩容(resize)，**即默认扩容阈值为12。**
  - 扩容倍数为：2倍
- DEFAULT_INITIAL_CAPACITY：默认初始容量为16
- table：存储Node的数组，链表状态下的节点。
  - length大小必须为2的n次方，减少hash冲突的现象。
- entrySet：存储entry的set集合
- size：实际存储的键值对数量
- modCount：对map结构操作的次数。
- TREEIFY_THRESHOLD：转为红黑树的链表节点阈值（条件之一）。
- MIN_TREEIFY_CAPACITY：树化的数组长度阈值（条件之一）。
- UNTREEIFY_THRESHOLD：树退化成链表的阈值。



```java
	//存储元素的数组，大小为2的幂次
	transient Node&lt;K,V&gt;[] table;

	//存放具体元素的集合
    transient Set&lt;Map.Entry&lt;K,V&gt;&gt; entrySet;

	//已经存放了的数组大小
    transient int size;

	//结构修改的计数器
    transient int modCount;

	//临界值，实际大小超过该值，则进行扩容
    int threshold;

	//负载因子
    final float loadFactor;

	//默认初始容量：16
	static final int DEFAULT_INITIAL_CAPACITY = 1 &lt;&lt; 4; // aka 16

	//最大容量
    static final int MAXIMUM_CAPACITY = 1 &lt;&lt; 30;

    //默认的负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    //链表节点数大于该阈值，转为红黑树存储
    static final int TREEIFY_THRESHOLD = 8;

    //红黑树节点数小于该值，转为链表存储
    static final int UNTREEIFY_THRESHOLD = 6;
	
	//树化时，检查table数组长度是否大于该值，小于则扩容
    static final int MIN_TREEIFY_CAPACITY = 64;
```



## 数据结构

链表状态下的节点，继承自`Map.Entry&lt;K,V&gt;`。

```java
static class Node&lt;K,V&gt; implements Map.Entry&lt;K,V&gt; {
    final int hash;
    final K key;
    V value;
    Node&lt;K,V&gt; next;

    //构造方法
    Node(int hash, K key, V value, Node&lt;K,V&gt; next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key &#43; &#34;=&#34; &#43; value; }

    //Node的HashCode返回键值的HashCode异或值
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
	//设置新值，返回旧值
    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
	//equals
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry&lt;?,?&gt; e = (Map.Entry&lt;?,?&gt;)o;
            if (Objects.equals(key, e.getKey()) &amp;&amp;
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}

//返回key的HashCode值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h &gt;&gt;&gt; 16);
}

//如果对象x的类是C，如果C实现了Comparable&lt;C&gt;接口，那么返回C，否则返回null
static Class&lt;?&gt; comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class&lt;?&gt; c; Type[] ts, as; Type t; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i &lt; ts.length; &#43;&#43;i) {
                if (((t = ts[i]) instanceof ParameterizedType) &amp;&amp;
                    ((p = (ParameterizedType)t).getRawType() ==
                     Comparable.class) &amp;&amp;
                    (as = p.getActualTypeArguments()) != null &amp;&amp;
                    as.length == 1 &amp;&amp; as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}
```



红黑树状态下的节点，继承自`LinkedHashMap.Entry&lt;K,V&gt;`，而该类继承自`HashMap.Node&lt;K,V&gt;`。

```java
static final class TreeNode&lt;K,V&gt; extends LinkedHashMap.Entry&lt;K,V&gt; {
        //父结点
    	TreeNode&lt;K,V&gt; parent;  // red-black tree links
    	//左孩子节点
        TreeNode&lt;K,V&gt; left;
    	//右孩子节点
        TreeNode&lt;K,V&gt; right;
        TreeNode&lt;K,V&gt; prev;    // needed to unlink next upon deletion
    	//颜色
        boolean red;
        TreeNode(int hash, K key, V val, Node&lt;K,V&gt; next) {
            super(hash, key, val, next);
        }

```



## 构造方法

核心的构造方法是第一个，通过调用`tableSizeFor`为`threshold`（扩容临界值）赋值。

&gt; Returns a power of two size for the given target capacity.


```java
	public HashMap(int initialCapacity, float loadFactor) {
        //边界检测
        if (initialCapacity &lt; 0)
            throw new IllegalArgumentException(&#34;Illegal initial capacity: &#34; &#43;
                                            initialCapacity);
        //边界检测
        if (initialCapacity &gt; MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor &lt;= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException(&#34;Illegal load factor: &#34; &#43;
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

	//调用前一个构造方法
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

   //未传参数时,负载因子设置为默认值
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }

    //传递Map的时候，调用putMapEntries进行批量添加
    public HashMap(Map&lt;? extends K, ? extends V&gt; m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```



## API


### tableSizeFor

该方法在初始化时，对`threshold`赋值，通过位运算找到大于或等于cap的最小的2的幂次方。

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n &gt;&gt;&gt; 1;
    n |= n &gt;&gt;&gt; 2;
    n |= n &gt;&gt;&gt; 4;
    n |= n &gt;&gt;&gt; 8;
    n |= n &gt;&gt;&gt; 16;
    return (n &lt; 0) ? 1 : (n &gt;= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n &#43; 1;
}
```



### hash（计算hash值）

该方法在插入时，计算元素的hash值时调用。`hash`值为传入key的`hashCode`与其右移16位的异或值。这被称为**扰动函数**。

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h &gt;&gt;&gt; 16);
}
```

`hashCode`方法，是key的类自带的hash方法，返回一个int的Hash值，理论上，这个值应当均匀得分布在int的范围内，但是`HashMap`初始化大小为16，如果让hash映射到16个桶中，通过取模实现十分简单。但是直接取模会造成较多的哈希碰撞。扰动函数的作用是：增加了随机性，减少了hash碰撞的几率。



那么如何通过hash值，获取对应的数组下标呢？在`putVal`中，获取下标的方法如下：`(n - 1) &amp; hash`，n是数组大小，上文中`threshold`为2&lt;sup&gt;n&lt;/sup&gt;，那么`(n - 1)`&lt;sub&gt;2&lt;/sub&gt; = 00....111111，那么通过`&amp;`运算，会保留下hash的低位。为何用`&amp;`替代取模运算，主要是位运算的效率高于取模运算。这也证明了为何`threshold`必须要是2的幂次方，通过控制该值，从而达到提高hash映射效率的目的。

```java
if ((p = tab[i = (n - 1) &amp; hash]) == null)
```



### get

主要逻辑：

1. 根据hash找到指定位置的节点
2. 判断第一个节点的key是否符合要求，符合要求直接返回第一个节点，否则继续查找。
3. 如果是红黑树结构，通过调用`getTreeNode(hash, key)`查找红黑树节点并返回。
4. 如果是链表结构，遍历节点查询并返回
5. 如果没有找到，返回null

```java
public V get(Object key) {
    Node&lt;K,V&gt; e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}


final Node&lt;K,V&gt; getNode(int hash, Object key) {
    Node&lt;K,V&gt;[] tab; Node&lt;K,V&gt; first, e; int n; K k;
    
    if ((tab = table) != null &amp;&amp; (n = tab.length) &gt; 0 &amp;&amp;
        (first = tab[(n - 1) &amp; hash]) != null) {
        //第一个节点就命中
        if (first.hash == hash &amp;&amp; // always check first node
            ((k = first.key) == key || (key != null &amp;&amp; key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            //判断是否为红黑树节点
            if (first instanceof TreeNode)
                return ((TreeNode&lt;K,V&gt;)first).getTreeNode(hash, key);
            //如果是链表，就遍历链表找到相应的数据
            do {
                if (e.hash == hash &amp;&amp;
                    ((k = e.key) == key || (key != null &amp;&amp; key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```



### put

主要逻辑：

1. 若桶数组table为空或length==0，则通过`resize()`进行扩容。
2. 根据key的hash得到数组索引，查找要插入的键值对是否已经存在，存在的话，用新值替换旧值。
3. 如果不存在，将键值对插入链表或者红黑树中，并根据长度判断是否将链表转换成红黑树。
4. 判断键值对数量是否大于阈值，大于的话进行扩容操作。

需要注意的是，`treeifyBin`方法在进行树化前，进行了检查。如果小于`MIN_TREEIFY_CAPACITY`，则进行扩容，不进行树化。

```java
if (tab == null || (n = tab.length) &lt; MIN_TREEIFY_CAPACITY)
```



```java
	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node&lt;K,V&gt;[] tab; Node&lt;K,V&gt; p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            //如果未初始化，进行数组初始化，赋予初始容量
            n = (tab = resize()).length;
    	//通过hash找到下标，如果该位置为空
    	//若下标处有节点存储，使用p存储
        if ((p = tab[i = (n - 1) &amp; hash]) == null)
            //直接将数据存储进去
            tab[i] = newNode(hash, key, value, null);
        else {
            //发生hash碰撞
            Node&lt;K,V&gt; e; K k;
            //如果插入的key和当前key相同，直接覆盖
            if (p.hash == hash &amp;&amp;
                ((k = p.key) == key || (key != null &amp;&amp; key.equals(k))))
                e = p;
            //如果当前节点类型是红黑树节点，使用红黑树进行插入
            else if (p instanceof TreeNode)
                e = ((TreeNode&lt;K,V&gt;)p).putTreeVal(this, tab, hash, key, value);
            else {
                //链表的情况
                for (int binCount = 0; ; &#43;&#43;binCount) {
                    //将新节点放到链表的末尾
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度达到红黑树化的阈值，将链表转化成红黑树
                        if (binCount &gt;= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果该key已经存在于链表中，覆盖
                    if (e.hash == hash &amp;&amp;
                        ((k = e.key) == key || (key != null &amp;&amp; key.equals(k))))
                        break;
                    p = e;
                }
            }
            //e不为空说明值已经插入成功
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent控制是否替换原来的value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        &#43;&#43;modCount;
    	//扩容检测
        if (&#43;&#43;size &gt; threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

//链表转为红黑树的方法
final void treeifyBin(Node&lt;K,V&gt;[] tab, int hash) {
        int n, index; Node&lt;K,V&gt; e;
    	//树化的第二个条件
        if (tab == null || (n = tab.length) &lt; MIN_TREEIFY_CAPACITY)
            //如果小于树化条件，使用resize
            resize();
        else if ((e = tab[index = (n - 1) &amp; hash]) != null) {
            TreeNode&lt;K,V&gt; hd = null, tl = null;
            do {
                TreeNode&lt;K,V&gt; p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```





### remove

主要逻辑：

1. 判断第一个节点是否是需删除节点，如果是，将节点存储下来。
2. 如果节点是红黑树节点，通过调用`getTreeNode`找到需删除节点，存储下来。
3. 如果是链表，遍历获取到需删除节点。
4. 删除节点，并进行修复工作

```java
public V remove(Object key) {
    Node&lt;K,V&gt; e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}


final Node&lt;K,V&gt; removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node&lt;K,V&gt;[] tab; Node&lt;K,V&gt; p; int n, index;
    if ((tab = table) != null &amp;&amp; (n = tab.length) &gt; 0 &amp;&amp;
        (p = tab[index = (n - 1) &amp; hash]) != null) {
        Node&lt;K,V&gt; node = null, e; K k; V v;
        //如果键与第一个节点相等，则该节点是需删除节点
        if (p.hash == hash &amp;&amp;
            ((k = p.key) == key || (key != null &amp;&amp; key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            //如果是红黑树节点，调用红黑树的查找方法找到需删除节点
            if (p instanceof TreeNode)
                node = ((TreeNode&lt;K,V&gt;)p).getTreeNode(hash, key);
            else {
                do {
                    //遍历链表，找到需删除节点
                    if (e.hash == hash &amp;&amp;
                        ((k = e.key) == key ||
                         (key != null &amp;&amp; key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        //删除节点，并修复链表或红黑树
        if (node != null &amp;&amp; (!matchValue || (v = node.value) == value ||
                             (value != null &amp;&amp; value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode&lt;K,V&gt;)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            &#43;&#43;modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```



### resize

`HashMap`的table数组长度为2的幂，阈值大小 = `capacity * load factor`（默认阈值为12），`Node`数量超过阈值，进行扩容，**扩容倍数为2倍。**

主要逻辑：

1. 计算新的桶数组容量`newCap`和新阈值`newThr`。`newCap`为原来的两倍，`newThr`为原来的两倍。
2. 根据`newCap`创建新的桶数组，初始化新的桶数组。
3. 将键值对节点重新映射到新的桶数组中，如果是红黑树节点，则需要拆分红黑树，如果是普通节点，则节点按照顺序进行分组。

需要注意的是：`resize`十分消耗性能，日常开发需要尽量避免。方法中变量含义如下：

- oldTab：引用扩容前的哈希表
- oldCap：表示扩容前的table数组的长度
- newCap：扩容之后table数组大小
- newThr：扩容之后下次触发扩容的阈值

```java
final Node&lt;K,V&gt;[] resize() {
    Node&lt;K,V&gt;[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //如果table长度大于0，说明已经被初始化
    if (oldCap &gt; 0) {
        //如果table的容量超过最大容量，不进行扩容
        if (oldCap &gt;= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //将容量变为原来的两倍，阈值变为原来的两倍
        else if ((newCap = oldCap &lt;&lt; 1) &lt; MAXIMUM_CAPACITY &amp;&amp;
                 oldCap &gt;= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr &lt;&lt; 1; // double threshold
    }
    //newCap = threshold
    else if (oldThr &gt; 0)
        newCap = oldThr;
    else {
        //阈值为默认容量与默认负载因子乘积
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //新阈值为0时，按照默认公式重新算newThr
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap &lt; MAXIMUM_CAPACITY &amp;&amp; ft &lt; (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //赋值
    threshold = newThr;
    @SuppressWarnings({&#34;rawtypes&#34;,&#34;unchecked&#34;})
    //创建新的桶数组
        Node&lt;K,V&gt;[] newTab = (Node&lt;K,V&gt;[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j &lt; oldCap; &#43;&#43;j) {
            Node&lt;K,V&gt; e;
            //如果旧的桶数组不为空，就遍历桶数组，并将键值对映射到新的桶数组
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash &amp; (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    //如果是红黑树节点，进行拆分
                    ((TreeNode&lt;K,V&gt;)e).split(this, newTab, j, oldCap);
                else { // 链表节点情况
                    Node&lt;K,V&gt; loHead = null, loTail = null;
                    Node&lt;K,V&gt; hiHead = null, hiTail = null;
                    Node&lt;K,V&gt; next;
                    do {
                        //遍历链表并进行按序分组
                        next = e.next;
                        if ((e.hash &amp; oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //将分组后的链表映射到新的桶数组中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j &#43; oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```



## 红黑树退化

在`TreeNode.split`方法和中，有一段代码对是否需要进行退化进行了判断。如果树节点个数小于`6`，则会退化为链表，至于该阈值与树化阈值`（UNTREEIFY_THRESHOLD 与 TREEIFY_THRESHOLD）`不等的原因，主要为了避免桶数组中的某个节点在该值附近震荡，从而导致频繁的树化和链表化。

```java
if (loHead != null) {
                if (lc &lt;= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc &lt;= UNTREEIFY_THRESHOLD)
                    tab[index &#43; bit] = hiHead.untreeify(map);
                else {
                    tab[index &#43; bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
```



此外，在`TreeNode.removeTreeNode`中，删除红黑树节点之前，如果满足以下条件，也会进行链表化再进行删除：

- 树的左子树为空
- 树的右子树为空
- 树的左孙子节点为空

```java
if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);  // too small
                return;
}
```



## 线程安全性

HashMap多线程操作存在以下问题：

- 多线程下扩容形成死循环：JDK1.7中使用头插法插入元素，扩容时可能导致形成环形链表，JDK1.8采用尾插法，不会出现此问题。
- 多线程put操作导致元素的丢失：多线程put时，发生hash碰撞，会导致key被覆盖，从而导致元素丢失。
- put和get并发，导致get为null：线程1执行put，因元素个数超出扩容阈值而导致resize，线程2执行get时，会得到null。





## 小结

本文对 JDK 8 中的 `HashMap` 的源代码进行了简要分析，主要为增删改查接口的内部实现机制以及扩容原理。

HashMap内部基于数组实现的，数组每个元素称为一个桶(bucket)，当存储的键值对数量超过阈值时，还会进行扩容操作，HashMap中的键值对会重新Hash到新位置。当桶中节点数超过阈值，则会进行树化，如果删除导致低于阈值，则会进行链表化。


---

> Author:   
> URL: http://localhost:1313/posts/java%E9%9B%86%E5%90%88/java-hashmap/  

