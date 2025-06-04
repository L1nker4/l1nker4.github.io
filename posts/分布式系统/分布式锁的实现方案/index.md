# 分布式锁的实现方案


## 简介

&gt; 分布式锁是控制分布式系统或不同系统之间共同访问共享资源的一种锁实现，如果不同的系统或同一个系统的不同主机之间共享了某个资源时，往往需要互斥来防止彼此干扰来保证一致性。

分布式锁需要具备的条件：

- 互斥性：在任意一个时刻，只有一个客户端持有锁。
- 无死锁：即使持有锁的客户端崩溃或者其他意外事件，锁仍然可以被获取。
- 容错：只要大部分Redis节点都活着，客户端可以获取和释放锁。

分布式锁的主要实现方式：

- 数据库
- Redis等缓存数据库，`redis`的`setnx`
- Zookeeper 临时节点

### Redis 实现分布式锁

### 第一阶段

```
public void hello() {
    Integer lock = get(&#34;lock&#34;);
    if(lock == null){
        set(&#34;lock&#34;,1);
        //todo
        del(&#34;lock&#34;);
        return;
    }else {
        //自旋
        hello();
    }
}



```

存在的问题：

- 请求同时打进来，所有请求都获取不到锁


### 第二阶段

使用`setnx`设置锁

`setnx`在key不存在时可以设置，存在当前key则不操作

```
public void hello(){
   String lock = setnx(&#34;lock&#34;); //0代表没有保存数据，说明有人占坑了，1代表占坑成功
    if(lock != 0){
        //todo
        del(&#34;lock&#34;);
    }else {
        //自旋
        hello();
    } 
}


```

存在的问题：

- 由于各种问题（未捕获的异常，断电）等导致锁未释放，其他人永远获取不到锁


### 第三阶段

```
public void hello(){
    String lock = setnx(&#34;lock&#34;);
    if(lock != 0){
        expire(&#34;lock&#34;,10s);
        //todo
        del(&#34;lock&#34;);
    }else {
        hello();
    }
}

```

存在的问题：

- 刚拿到锁，服务器出现问题，没来得及设置过期时间


### 第四阶段

```
public void hello() {
    String result = setnxex(&#34;lock&#34;,10s);
    if(result == &#34;ok&#34;){
        //加锁成功
        //执行业务逻辑
        //释放锁
    }else {
        hello();
    }
}

```

存在的问题：

- 如果业务逻辑超时，导致锁自动删除，业务执行完又删除了一遍，至少多个人都获取到了锁。


### 第五阶段

```
public void hello() {
    String token = UUID;
    String result = setnxex(&#34;lock&#34;,token,10s);
    if(result == &#34;ok&#34;){
        //执行业务
        //释放锁,保证删除自己的锁
        get(&#34;lock&#34;) == token ? del(&#34;lock&#34;) : return;
    }
}

```

存在的问题：

- 获取锁的时候，锁的值正在返回的过程中，锁过期，redis删除了锁，但是我们拿到了值，对比成功（此时另一个请求创建了锁），但是该线程又删除了一次锁。

原因：

- 删锁不是原子性的
- 解决：`Lua`脚本


### 第六阶段

```
public void hello() {
    String token = UUID;
    String lock = redis.setnxex(&#34;lock&#34;,token,10s);
    if(lock == &#34;ok&#34;){
        //执行业务
        //脚本删除锁
    }else {
        hello();
    }
}

```






## Redisson实现分布式锁

```
public void hello() {
    Rlock lock = redisson.getLock(&#34;lock&#34;);
    lock.lock();
    //todo
    lock.unlock();
}

```



## 其他方向考虑

- 自旋
  - 自旋次数
  - 自旋超时
- 原子的加锁释放没问题
- 锁设置
  - 锁粒度：细，记录级别
    - 查询商品详情：进缓存-&gt;击穿，穿透，雪崩，不同ID的商品不同的锁
    - 分析好粒度，不要锁住无关数据，一种数据一种锁，一条数据一个锁
- 锁类型
  - 读写锁
  - 可重入锁



---

> Author:   
> URL: http://localhost:1313/posts/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E6%A1%88/  

