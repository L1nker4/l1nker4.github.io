# TreeMap源码分析






## 简介

`TreeMap`底层通过红黑树实现，在查询性能上能达到`O(logn)`，由于使用红黑树结构进行存储，所以`TreeMap`的元素都是有序的。同时，这也是一个非线程安全的`Map`，无法在并发环境下使用。



`TreeMap`继承自`AbstractMap`，该类Map接口的抽象实现。实现了 `NavigableMap`、`Cloneable`和 `Serializable`接口。其中`NavigableMap`继承自`SortedMap`，这保证了`TreeMap`的有序性。

```java
public class TreeMap&lt;K,V&gt;
    extends AbstractMap&lt;K,V&gt;
    implements NavigableMap&lt;K,V&gt;, Cloneable, java.io.Serializable
```



## 数据结构

`TreeMap`采用红黑树进行构建，红黑树是一种自平衡二叉查找树。插入、删除、查找的时间复杂度为`O(logn)`。与另一个自平衡二叉查找树`AVL Tree`相比，红黑树以减少旋转操作牺牲部分平衡性，但是其整体性能优于`AVL Tree`。

有关红黑树的定义如下（摘自wikipedia）：

1. 节点是红色或黑色。
2. 根是黑色。
3. 所有叶子都是黑色（叶子是NIL节点）。
4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

![红黑树结构示意图（摘自Wikipedia）](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/java/treemap/rbtree-construction.png)





`TreeMap`中树节点的定义如下：



```java
static final class Entry&lt;K,V&gt; implements Map.Entry&lt;K,V&gt; {
        K key;
        V value;
        Entry&lt;K,V&gt; left;
        Entry&lt;K,V&gt; right;
        Entry&lt;K,V&gt; parent;
        boolean color = BLACK;

        
        Entry(K key, V value, Entry&lt;K,V&gt; parent) {
            this.key = key;
            this.value = value;
            this.parent = parent;
        }

        
        public K getKey() {
            return key;
        }

        
        public V getValue() {
            return value;
        }

    
        public V setValue(V value) {
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry&lt;?,?&gt; e = (Map.Entry&lt;?,?&gt;)o;

            return valEquals(key,e.getKey()) &amp;&amp; valEquals(value,e.getValue());
        }

        public int hashCode() {
            int keyHash = (key==null ? 0 : key.hashCode());
            int valueHash = (value==null ? 0 : value.hashCode());
            return keyHash ^ valueHash;
        }

        public String toString() {
            return key &#43; &#34;=&#34; &#43; value;
        }
    }
```



## 成员变量



```java
	//TreeMap中用来确定顺序的comparator
	private final Comparator&lt;? super K&gt; comparator;

	//树的根节点
    private transient Entry&lt;K,V&gt; root;

    //树的大小
    private transient int size = 0;

    //结构变化计数器
    private transient int modCount = 0;

	//EntrySet
	private transient EntrySet entrySet;
    private transient KeySet&lt;K&gt; navigableKeySet;
    private transient NavigableMap&lt;K,V&gt; descendingMap;

	//SubMapIterator中fence == null时，key的值
	private static final Object UNBOUNDED = new Object();

	//RB Tree的颜色变量
	private static final boolean RED   = false;
    private static final boolean BLACK = true;
```





## 构造方法

共有四个构造方法：

```java
public TreeMap() {
        comparator = null;
    }

    //传入comparator
    public TreeMap(Comparator&lt;? super K&gt; comparator) {
        this.comparator = comparator;
    }

    //传入map
    public TreeMap(Map&lt;? extends K, ? extends V&gt; m) {
        comparator = null;
        putAll(m);
    }

   //传入有序map
    public TreeMap(SortedMap&lt;K, ? extends V&gt; m) {
        comparator = m.comparator();
        try {
            buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
    }
```



## 基本操作



### 左旋

![左旋操作](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/java/treemap/rbtree-rotateLeft.png)



上图中的5失衡，需要对该节点进行左旋进行修复。

```java
/** From CLR */
    private void rotateLeft(Entry&lt;K,V&gt; p) {
        if (p != null) {
            //r为右子树节点
            Entry&lt;K,V&gt; r = p.right;
           //p的右子树换成它子的左孩子节点
            p.right = r.left;
            //如果p的右孩子节点不为空
            if (r.left != null)
                //将该节点的父节指向p
                r.left.parent = p;
            //r的父节点指向p的父结点
            r.parent = p.parent;
            //判断旋转的p节点是否为树的根节点
            if (p.parent == null)
                //如果是，将根节点设置为r
                root = r;
            //如果失衡节点p是父节点的左孩子节点
            else if (p.parent.left == p)
                //将父节点的左孩子节点设置为r
                p.parent.left = r;
            else
                //失衡节点是父节点的右孩子节点
                p.parent.right = r;
            //将r节点的左子树设置为失衡节点p
            r.left = p;
            p.parent = r;
        }
    }
```





### 右旋



![右旋操作](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/java/treemap/rbtree-rorateRight.jpg)



上图中的10失衡，需要对该节点进行右旋进行修复。

```java
private void rotateRight(Entry&lt;K,V&gt; p) {
    	//失衡节点传入不为空
        if (p != null) {
           	//l为失衡节点的左子树
            Entry&lt;K,V&gt; l = p.left;
            //将失衡节点的左孩子节点指向它左子树的左孩子节点
            p.left = l.right;
            //l的右子树不为空，将右子树的父指针指向p
            if (l.right != null) l.right.parent = p;
            //l升级为原来p节点的地位
            l.parent = p.parent;
            //如果原来的p节点为根节点，将l设置为根节点
            if (p.parent == null)
                root = l;
            //如果p是父节点的右孩子，则将其父节点的右孩子设置为l
            else if (p.parent.right == p)
                p.parent.right = l;
            else p.parent.left = l;
            //设置l的右节点为p，右旋完成
            l.right = p;
            p.parent = l;
        }
    }
```



## API



### get

`get`通过调用`getEntry`获取对应的`entry`。返回的是`entry.value`。查找逻辑较为简单，是`BST`经典查询代码。代码注释如下：

```java
public V get(Object key) {
        Entry&lt;K,V&gt; p = getEntry(key);
        return (p==null ? null : p.value);
}

final Entry&lt;K,V&gt; getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key);
    	//不允许key为空
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings(&#34;unchecked&#34;)
            Comparable&lt;? super K&gt; k = (Comparable&lt;? super K&gt;) key;
        Entry&lt;K,V&gt; p = root;
    	//从根节点开始找
        while (p != null) {
            
            int cmp = k.compareTo(p.key);
            //比key大，往左子树找
            if (cmp &lt; 0)
                p = p.left;
            //比key小，往右子树找
            else if (cmp &gt; 0)
                p = p.right;
            else
                return p;
        }
        return null;
}
```



### put

`put`方法首先检查是否已经存在该`key`，如果有则覆盖，没有则构造新节点进行插入。插入后调用`fixAfterInsertion`进行红黑树的修复。

```java
public V put(K key, V value) {
        Entry&lt;K,V&gt; t = root;
    	//根节点为空
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new Entry&lt;&gt;(key, value, null);
            size = 1;
            modCount&#43;&#43;;
            return null;
        }
        int cmp;
        Entry&lt;K,V&gt; parent;
        // split comparator and comparable paths
        Comparator&lt;? super K&gt; cpr = comparator;
    	//comparator不为空
        if (cpr != null) {
            //遍历找到与该key相等的节点，覆盖旧值
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp &lt; 0)
                    t = t.left;
                else if (cmp &gt; 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
            //comparator为空
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings(&#34;unchecked&#34;)
                Comparable&lt;? super K&gt; k = (Comparable&lt;? super K&gt;) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp &lt; 0)
                    t = t.left;
                else if (cmp &gt; 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
    	//没有相同的key，构造新Entry进行插入
        Entry&lt;K,V&gt; e = new Entry&lt;&gt;(key, value, parent);
        if (cmp &lt; 0)
            parent.left = e;
        else
            parent.right = e;
    	//插入后修复
        fixAfterInsertion(e);
        size&#43;&#43;;
        modCount&#43;&#43;;
        return null;
    }
```



### remove

`remove`方法首先调用`getEntry`获取需要删除的entry，调用`deleteEntry`进行删除。红黑树的删除逻辑与二叉查找树相类似，可以分为两种情况：

1. 待删除节点P的左右子树都为空，则直接删除
2. 待删除节点P的左右子树非空，用P的后继节点代替P

```java
public V remove(Object key) {
        Entry&lt;K,V&gt; p = getEntry(key);
        if (p == null)
            return null;

        V oldValue = p.value;
        deleteEntry(p);
        return oldValue;
    }

private void deleteEntry(Entry&lt;K,V&gt; p) {
        modCount&#43;&#43;;
        size--;

        //上述的第二种情况，找到P的后继节点代替它
        if (p.left != null &amp;&amp; p.right != null) {
            Entry&lt;K,V&gt; s = successor(p);
            p.key = s.key;
            p.value = s.value;
            p = s;
        } // p has 2 children

        // Start fixup at replacement node, if it exists.
        Entry&lt;K,V&gt; replacement为 = (p.left != null ? p.left : p.right);
		//replacement用来代替删除节点
        if (replacement != null) {
            // Link replacement to parent
            replacement.parent = p.parent;
            //p没有父节点
            if (p.parent == null)
                //说明它是根节点，直接将replacement设置为根节点。
                root = replacement;
            else if (p == p.parent.left)
                p.parent.left  = replacement;
            else
                p.parent.right = replacement;

            // Null out links so they are OK to use by fixAfterDeletion.
            p.left = p.right = p.parent = null;

            // 进行删除后修复
            if (p.color == BLACK)
                fixAfterDeletion(replacement);
        } else if (p.parent == null) { // return if we are the only node.
            root = null;
        } else { //  No children. Use self as phantom replacement and unlink.
            if (p.color == BLACK)
                fixAfterDeletion(p);

            if (p.parent != null) {
                if (p == p.parent.left)
                    p.parent.left = null;
                else if (p == p.parent.right)
                    p.parent.right = null;
                p.parent = null;
            }
        }
    }
```



## 总结

本文介绍了`TreeMap`的数据结构上的实现，并介绍了红黑树的基本概念，并对增删改查的接口做了简要介绍，但是并未深入探究修复的接口（`fixAfterDeletion`和`fixAfterInsertion`）。

---

> Author:   
> URL: http://localhost:1313/posts/java%E9%9B%86%E5%90%88/java-treemap/  

