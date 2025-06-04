# Netty学习笔记(二)- 内部机制




# 事件调度层



## Reactor线程模型



Netty中三种Reactor线程模型来源于[Scalable I/O in Java](https://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)，主要有以下三种：
- 单线程模型：所有IO操作（连接建立、读写、事件分发）都由一个线程完成。
- 多线程模型：使用多线程处理任务。线程内部仍然是串行化。
- 主从多线程模型：主线程只负责连接的Accept事件，从线程负责除连接外的事件。



Reactor线程模型运行机制可以分为以下四个步骤：

- 注册连接：将Channel注册到Reactor线程中的Selector。
- 事件轮询：轮询Selector中已注册的Channel的IO事件。
- 事件分发：将连接的IO事件分发给worker线程。
- 任务处理：Reactor线程负责队列中的非IO任务。



## EventLoop


EventLoop是一种**事件处理模型**，Netty中EventLoop运行机制如下图所示：

![EventLoop运行机制](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20221113182321659.png)

- BossEventLoopGroup：负责监听客户端的Accept事件，触发时将事件注册到WorkerEventLoopGroup中的一个NioEventLoop，
- WorkerEventLoopGroup：每建立一个Channel，都选择一个NioEventLoop与其绑定，Channel的所有事件都是线程独立的。不会和其他线程发生交集。



### 任务处理机制

NioEventLoop不仅负责处理IO事件，还要兼顾执行任务队列中的任务。任务队列遵守FIFO原则。任务基本可以分为三类：

- 普通任务：通过NioEventLoop的execute()方法向taskQueue中添加的。
- 定时任务：通过NioEventLoop的schedule()方法向scheduledtaskQueue添加的定时任务，例如心跳消息可以通过该任务实现。
- 尾部队列：执行完taskQueue中任务后会去获取尾部队列tailTasks中的任务去执行。主要做收尾工作，例如统计事件循环的执行时间等。





### 使用技巧

- 使用Boss和Worker两个Group分别处理不同的事件，合理分担压力。
- 对于耗时较长的ChannelHandler可以考虑维护一个业务线程池，将任务封装成Task进行异步处理。，避免ChannelHandler阻塞而造成EventLoop不可用。
- 选用合理的ChannelHandler数量，明确业务和Netty的分层。



# 服务编排层



## ChannelPipeline

Pipeline如同字面意思，原始的网络字节流流经pipeline，被逐层处理，最终得到成品数据并返回。Netty中的ChannelPipeline采取责任链模式，调用链路环环相扣。



ChannelPipeline由一组ChannelHandlerContext组成，内部通过双向链表将ChannelHandlerContext连接起来，当IO事件触发时，依次调用Handler对数据进行处理。ChannelHandlerContext用于保存ChannelHandler的上下文，包含了其生命周期的所有事件：connect、bind、read等。



根据数据流向，ChannelPipeline可以分为入站和出站两种处理器，对应**ChannelInboundHandler**和**ChannelOutboundHandler**。



## 异常处理

ChannelHandler采用了责任链模式，如果前置的Handler抛出呢Exception，会传递到后置Handler，异常处理的最佳实践，就是在最后加上自定义的异常处理器。

```java
public class ExceptionHandler extends ChannelDuplexHandler {

    @Override

    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {

        if (cause instanceof RuntimeException) {

            System.out.println(&#34;Handle Business Exception Success.&#34;);
        }
    }

}
```



# 零拷贝

Netty中面向用户态的数据操作优化，主要包含以下几个方面：

1. 使用堆外内存，避免JVM内存到堆外内存之间的数据拷贝
2. 使用CompositeByteBuf，可以组合多个Buffer，将其合并成逻辑上的一个对象，避免物理的内存拷贝。
3. 使用Unpooled.wrappedBuffer，将byte数组包装成ByteBuf对象，过程间不产生内存拷贝。
4. ByteBuf.slice切分时不产生内存拷贝，底层共享一个byte数组。
5. 使用FileRegion实现文件传输，使用的FileChannel#transferTo()，直接将缓冲区数据输出到目标Channel，避免内核缓冲区和用户态缓冲区的数据拷贝。







---

> Author:   
> URL: http://localhost:1313/posts/netty/netty%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%BA%8C--%E5%86%85%E9%83%A8%E6%9C%BA%E5%88%B6/  

