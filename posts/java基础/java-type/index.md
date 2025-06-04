# Integer小细节




## Integer



### IntegerCache 

在Java中，创建的对象会存储在堆内存，·IntegerCache·主要用来做缓存，减少内存的重复消耗。但是`IntegerCache`缓存的数据范围在-128到127之间。可以通过 `-XX:AutoBoxCacheMax=high` 来指定这个缓冲池的大小，JVM通过` java.lang.IntegerCache.high` 系统属性来存储该值，` IntegerCache `初始化的时候就会读取该系统属性来决定大小。以下为源码：

```java
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            //读取JVM参数值
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty(&#34;java.lang.Integer.IntegerCache.high&#34;);
            //初始化high
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;
            //127 - (-128) &#43; 1为数组容量
            cache = new Integer[(high - low) &#43; 1];
            //初始化Cache数组
            int j = low;
            for(int k = 0; k &lt; cache.length; k&#43;&#43;)
                cache[k] = new Integer(j&#43;&#43;);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high &gt;= 127;
        }

        private IntegerCache() {}
    }
```



### Integer.valueOf()

首先看一下关于该方法的实现：

```java
public static Integer valueOf(int i) {
    	//判断传入的变量是否处于IntegerCache之内，是的话直接返回缓存中的数据
        if (i &gt;= IntegerCache.low &amp;&amp; i &lt;= IntegerCache.high)
            return IntegerCache.cache[i &#43; (-IntegerCache.low)];
        return new Integer(i);
    }

public Integer(int value) {
        this.value = value;
}
```



因此以下代码便十分容易理解：

- 构造方法是创建一个新对象，必然不是同一个地址
- `valueOf`在`-128 ~ 127`区间之内返回`IntegerCache`中的对应数组的元素。而超过该范围通过构造方法进行创建。

```java
Integer x = new Integer(123);
Integer y = new Integer(123);
// false
System.out.println(x == y);    
Integer a = Integer.valueOf(123);
Integer b = Integer.valueOf(123);
// true
System.out.println(a == b);
Integer c = Integer.valueOf(129);
Integer d = Integer.valueOf(129);
//false
System.out.println(c == d);
```




---

> Author:   
> URL: http://localhost:1313/posts/java%E5%9F%BA%E7%A1%80/java-type/  

