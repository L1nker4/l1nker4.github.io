# 解析Java动态代理机制的实现




## 简介

代理模式主要是Proxy对原始方法做了一层包装，用以增加一些新的统一处理逻辑，来增强目标对象的功能。静态代理是传统设计模式中一种传统的实现方案，动态代理能将代理对象的创建延迟到程序运行阶段。



以下是一个动态代理的示例：

被代理类：

```java
public interface DemoService {

    public String process(String value);
}


@Slf4j
public class DemoServiceImpl implements DemoService{

    public static final String RESULT = &#34;result&#34;;

    @Override
    public String process(String value) {
        log.info(&#34;{} is processing, parameter: {}&#34;, this.getClass().getName(), value);
        return RESULT;
    }
}


```



代理类：

```java
@Slf4j
public class DemoInvokerHandler implements InvocationHandler {

    private final Object target;

    public DemoInvokerHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log.info(&#34;before...&#34;);
        Object result = method.invoke(target, args);
        log.info(&#34;after...&#34;);
        return result;
    }

    public Object getProxy() {
        // 创建代理对象
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                target.getClass().getInterfaces(), this);

    }
}
```



代理调用：

```java
@Slf4j
public class ProxyMain {

    public static void main(String[] args) {
        DemoService service = new DemoServiceImpl();
        DemoInvokerHandler demoInvokerHandler = new DemoInvokerHandler(service);
        DemoService proxy = (DemoService) demoInvokerHandler.getProxy();
        String result = proxy.process(&#34;test&#34;);
        log.info(&#34;result is : {}&#34;, result);
    }
}
```





## 动态代理实现原理



通过上述示例中`getProxy()`方法可以看出，创建代理对象的核心方法是：

```
@CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class&lt;?&gt;[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
```

传入参数分别为：

- loader：加载动态代理类的classloader
- interfaces：代理目标的接口类型
- h：InvocationHandler类型，用于执行代理目标的方法时，触发的handler



该方法的具体实现与注释如下：

```java
{
        Objects.requireNonNull(h);

    	//检查classloader、包权限等
        final Class&lt;?&gt;[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
    	//获取代理类class
        Class&lt;?&gt; cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            //获取代理类的构造方法
            final Constructor&lt;?&gt; cons = cl.getConstructor(constructorParams);
            //Q：这里赋值ih的意义是什么
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction&lt;Void&gt;() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            //创建代理对象
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```



加载Class对象是通过`getProxyClass0()`方法实现的：

```java
private static Class&lt;?&gt; getProxyClass0(ClassLoader loader,
                                           Class&lt;?&gt;... interfaces) {
        if (interfaces.length &gt; 65535) {
            throw new IllegalArgumentException(&#34;interface limit exceeded&#34;);
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```

`proxyClassCache`用于缓存已经创建过的代理类，定义如下：

```java
private static final WeakCache&lt;ClassLoader, Class&lt;?&gt;[], Class&lt;?&gt;&gt;
        proxyClassCache = new WeakCache&lt;&gt;(new KeyFactory(), new ProxyClassFactory());
```



### 代理类缓存WeakCache

WeakCache是上述缓存proxyClassCache的具体实现，WeakCache包含以下字段：

- refQueue：ReferenceQueue类型，用于存储被回收的key
  - ReferenceQueue：用于处理弱引用、软引用和虚引用的回收操作，当对象被GC后，可以从queue中获取对应的对象信息，进行其他处理。
- map：ConcurrentMap类型，存储键值对的map，有两级缓存，存储结构为：key -&gt; (subKey -&gt; value)
  - 一级缓存的key为classloader
  - 二级缓存为KeyFactory为代理类的接口数组生成的key，包括key0、key1、key2、keyx

- reverseMap：记录了所有可用的代理类，key为代理类的cacheValue
- subKeyFactory：BiFunction函数式接口，生成subKey的工厂类，Proxy类传入：`java.lang.reflect.Proxy.KeyFactory`
- valueFactory：BiFunction函数式接口，生成value的工厂类，Proxy类传入：`java.lang.reflect.Proxy.ProxyClassFactory`

```java
 /*
 * @author Peter Levart
 * @param &lt;K&gt; type of keys
 * @param &lt;P&gt; type of parameters
 * @param &lt;V&gt; type of values
 */
final class WeakCache&lt;K, P, V&gt; {

    private final ReferenceQueue&lt;K&gt; refQueue
        = new ReferenceQueue&lt;&gt;();
    // the key type is Object for supporting null key
    private final ConcurrentMap&lt;Object, ConcurrentMap&lt;Object, Supplier&lt;V&gt;&gt;&gt; map
        = new ConcurrentHashMap&lt;&gt;();
    private final ConcurrentMap&lt;Supplier&lt;V&gt;, Boolean&gt; reverseMap
        = new ConcurrentHashMap&lt;&gt;();
    private final BiFunction&lt;K, P, ?&gt; subKeyFactory;
    private final BiFunction&lt;K, P, V&gt; valueFactory;
}
```



#### KeyFactory

`java.lang.reflect.Proxy.KeyFactory`是用于生成二级cache的subKey的工厂类，返回的对象为`WeakReference`

```java
private static final class KeyFactory
        implements BiFunction&lt;ClassLoader, Class&lt;?&gt;[], Object&gt;
    {
        @Override
        public Object apply(ClassLoader classLoader, Class&lt;?&gt;[] interfaces) {
            switch (interfaces.length) {
                case 1: return new Key1(interfaces[0]); // the most frequent
                case 2: return new Key2(interfaces[0], interfaces[1]);
                case 0: return key0;
                default: return new KeyX(interfaces);
            }
        }
    }


private static final class Key1 extends WeakReference&lt;Class&lt;?&gt;&gt; {}
    
private static final class Key2 extends WeakReference&lt;Class&lt;?&gt;&gt; {}

private static final class KeyX {
        private final int hash;
        private final WeakReference&lt;Class&lt;?&gt;&gt;[] refs;
}
```





#### ProxyClassFactory

`java.lang.reflect.Proxy.ProxyClassFactory`根据传入classloader和接口类数组，返回代理类的工厂。主要流程如下：

- 校验参数的正确性：
  - 校验classloader加载的接口类和传入接口是否为同一个
  - 校验是否为接口类
  - 校验接口是否重复
  - 判断接口的权限修饰符，非public接口的代理类包名取接口类的包名，public取`com.sun.proxy`
- 设置代理类的包名：包名 &#43; $Proxy &#43; 原子递增数字
  - 例如：`com.sun.proxy.$Proxy0`
- 调用`ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags)`生成class文件
- 调用native方法`defineClass0()`将class文件加载到JVM







#### WeakCache#get

`java.lang.reflect.Proxy#getProxyClass0`中调用`get(loader, interfaces)`，传入的key为`ClassLoader`类型，parameter为`Class&lt;?&gt;`类型，为代理类接口。主要流程如下：

- 参数校验、执行`expungeStaleEntries()`清理过期对象
- 使用一级cache的key获取二级cache的map对象
- 向KeyFactory传入key和parameter，生成二级cache的key
- 轮询获取supplier中的value
  - 如果supplier为空，将当前subKey对应的数据设置为Factory实例。
  - Factory会通过ProxyClassFactory生成代理类

```java
public V get(K key, P parameter) {
        Objects.requireNonNull(parameter);

        expungeStaleEntries();
		//包装classloader，生成一级cache的key
        Object cacheKey = CacheKey.valueOf(key, refQueue);

        // lazily install the 2nd level valuesMap for the particular cacheKey
    	//获取二级cache的map对象
        ConcurrentMap&lt;Object, Supplier&lt;V&gt;&gt; valuesMap = map.get(cacheKey);
        if (valuesMap == null) {
            ConcurrentMap&lt;Object, Supplier&lt;V&gt;&gt; oldValuesMap
                = map.putIfAbsent(cacheKey,
                                  valuesMap = new ConcurrentHashMap&lt;&gt;());
            if (oldValuesMap != null) {
                valuesMap = oldValuesMap;
            }
        }

        // create subKey and retrieve the possible Supplier&lt;V&gt; stored by that
        // subKey from valuesMap
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    	//获取二级cache的值
        Supplier&lt;V&gt; supplier = valuesMap.get(subKey);
        Factory factory = null;
		//轮询获取supplier中的value
        while (true) {
            if (supplier != null) {
                // supplier might be a Factory or a CacheValue&lt;V&gt; instance
                V value = supplier.get();
                if (value != null) {
                    return value;
                }
            }
            // else no supplier in cache
            // or a supplier that returned null (could be a cleared CacheValue
            // or a Factory that wasn&#39;t successful in installing the CacheValue)

            // lazily construct a Factory
            if (factory == null) {
                factory = new Factory(key, parameter, subKey, valuesMap);
            }
			//二级cache直接放入subKey -&gt; factory的映射
            //设置supplier = factory
            if (supplier == null) {
                supplier = valuesMap.putIfAbsent(subKey, factory);
                if (supplier == null) {
                    // successfully installed Factory
                    supplier = factory;
                }
                // else retry with winning supplier
            } else {
                //轮询时如果其他线程已经设置了值的情况
                //替换为factory
                if (valuesMap.replace(subKey, supplier, factory)) {
                    // successfully replaced
                    // cleared CacheEntry / unsuccessful Factory
                    // with our Factory
                    supplier = factory;
                } else {
                    // retry with current supplier
                    supplier = valuesMap.get(subKey);
                }
            }
        }
    }
```



#### Factory

通过Factory实例调用`ProxyClassFactory.apply(key, parameter)`生成代理类，通过`synchronized`保证线程安全，get()的具体逻辑参考注释：

```java
private final class Factory implements Supplier&lt;V&gt; {
		//一级cache的key，为classloader
        private final K key;
    	//代理类的接口数组
        private final P parameter;
    	//二级cache的key，根据接口数组生成
        private final Object subKey;
    	//二级缓存的引用
        private final ConcurrentMap&lt;Object, Supplier&lt;V&gt;&gt; valuesMap;

        Factory(K key, P parameter, Object subKey,
                ConcurrentMap&lt;Object, Supplier&lt;V&gt;&gt; valuesMap) {
            this.key = key;
            this.parameter = parameter;
            this.subKey = subKey;
            this.valuesMap = valuesMap;
        }

        @Override
        public synchronized V get() { // serialize access
            // re-check
            Supplier&lt;V&gt; supplier = valuesMap.get(subKey);
            //避免多线程数据不一致
            if (supplier != this) {
                // something changed while we were waiting:
                // might be that we were replaced by a CacheValue
                // or were removed because of failure -&gt;
                // return null to signal WeakCache.get() to retry
                // the loop
                return null;
            }
            // else still us (supplier == this)

            // create new value
            V value = null;
            try {
                //通过ProxyClassFactory去生成代理类
                value = Objects.requireNonNull(valueFactory.apply(key, parameter));
            } finally {
                if (value == null) { // remove us on failure
                    valuesMap.remove(subKey, this);
                }
            }
            // the only path to reach here is with non-null value
            assert value != null;

            // wrap value with CacheValue (WeakReference)
            CacheValue&lt;V&gt; cacheValue = new CacheValue&lt;&gt;(value);

            // put into reverseMap
            reverseMap.put(cacheValue, Boolean.TRUE);

            // try replacing us with CacheValue (this should always succeed)
            if (!valuesMap.replace(subKey, this, cacheValue)) {
                throw new AssertionError(&#34;Should not reach here&#34;);
            }

            // successfully replaced us with new CacheValue -&gt; return the value
            // wrapped by it
            return value;
        }
    }
```









---

> Author:   
> URL: http://localhost:1313/posts/java%E5%9F%BA%E7%A1%80/%E8%A7%A3%E6%9E%90java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E6%9C%BA%E5%88%B6/  

