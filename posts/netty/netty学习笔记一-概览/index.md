# Netty学习笔记(一)-概览


## 简介

Netty是一个应用于网络编程领域的NIO网络框架，通过屏蔽底层Socket编程细节，封装了提供上层业务使用的API，简化了网络应用的开发过程。Netty需要关注以下几点：

- IO模型、线程模型
- 事件处理机制
- API接口的使用
- 数据协议、序列化的支持

Netty的IO模型是基于非阻塞IO实现的，底层通过`JDK NIO`中的`Selector`实现，`Selector`可以同时轮询多个`Channel`，采用`epoll`模式后只需要一个线程负责`Selector`的轮询。



IO多路复用的场景中，需要一个`Event Dispather`负责将读写事件分发给对应的`Event Handler`，事件分发器主要有两种：

- Reactor：采用同步IO，实现简单，适用于处理耗时短的场景，耗时长的IO操作容易出现阻塞。
- Proactor：采用异步IO，实现逻辑复杂，性能更高



Netty的优点：

- 易用：将NIO的API进一步封装，提供了开箱即用的工具
- 稳定：修复了NIO的bug
- 可扩展：可以通过启动参数选择Reactor线程模型
- 低消耗：Netty性能优化
  - 对象池复用
  - 零拷贝



## NIO基础

NIO是一种同步非阻塞的IO模型，NIO与普通IO的最大区别就是非阻塞，通过每个线程通过Selector去监听多个Channel，并且读写数据是以块为单位，与BIO相比，大大提升了IO效率。

BIO存在的问题：

- accept、read、write都是同步阻塞，处理IO时，线程阻塞。
- BIO模型严重依赖线程，线程资源比较宝贵。
  - Linux中用`task_struct`管理，创建或销毁线程使用系统调用，开销大，并且进程切换也存在开销
  - 每个线程在JVM中占用1MB内存，连接数量大的时候，极易产生OOM



Standard IO是对字节流进行读写，读写单位是字节，NIO将IO抽象成块，读写单位是块。

基本概念：

- Channel：对原IO包中流的模拟，可以通过它来读取和写入数据，数据流向是双向的。

  - FileChannel：从文件中读取数据
  - DatagramChannel：通过UDP读写网络数据
  - SocketChannel：通过TCP读写网络数据
  - ServerSocketChannel：监听新的TCP连接，对每个新连接都创建一个SocketChannel
- Buffer：Channel中的数据都需要通过Buffer进行传递，本质上是数组

  - ByteBuffer、CharBuffer等
  - Buffer的内部变量：
    - capacity：最大容量
    - position：当前读写处的下标位置
    - limit：还可读写的下标位置
- Selector：NIO采用的Reactor模型，一个线程使用一个Selector通过轮询的方式去监听多个Channel上面的事件，将Channel配置为非阻塞，那么Selector检测到当前Channel没有IO事件，就会轮询其他Channel。



内存映射文件：是一种读写文件的方式，比常规基于流或者Channel的IO快。

```java
//将文件的前1024字节映射到内存中，map()方法返回一个MappedByteBuffer
MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, 0, 1024);
```



## Demo

```java
public class SimpleServer {
    public static void main(String[] args) {
        /**
         * ServerBootstrap：服务端启动器，负责组装、协调netty组件
         * NioEventLoopGroup：thread &#43; selector
         * NioServerSocketChannel：对原生NIO的ServerSocketChannel封装
         * ChannelInitializer：对channel进行初始化
         */
        new ServerBootstrap()
                .group(new NioEventLoopGroup())
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer&lt;NioSocketChannel&gt;() {
                    //连接建立后执行initChannel
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        //StringDecoder：将Bytebuffer转为string
                        ch.pipeline().addLast(new StringDecoder());
                        //自定义handler
                        ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                System.out.println(msg);
                            }
                        });
                    }
                })
                .bind(8080);
    }
}

public class SimpleClient {
    public static void main(String[] args) throws IOException, InterruptedException {
        new Bootstrap()
                .group(new NioEventLoopGroup())
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer&lt;NioSocketChannel&gt;() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new StringEncoder());
                    }
                })
                .connect(new InetSocketAddress(&#34;localhost&#34;, 8080))
                //阻塞直到连接建立
                .sync()
                //代表连接对象
                .channel()
                //发送数据
                .writeAndFlush(&#34;hello world&#34;);
    }
}
```





## Netty组件

- Core：提供了底层网络通信的抽象和实现，支持零拷贝的ByteBuffer、可扩展的事件模型、通信API。
- 协议支持层：对主流协议的编解码实现，包括：HTTP、SSL、Protobuf等，还支持自定义应用层协议。
- 传输服务层：提供了网络传输能力的抽象和实现，支持Socket、HTTP tunnel、VM pipe等方式



![netty-components](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/components.png)



从逻辑架构上可以分为：

- 网络通信层：执行网络IO的操作，支持多种网络协议和IO模型，数据被读取到内核缓冲区时，会触发各种网络事件，事件会分发给上层处理，包括以下组件：
  - Bootstrap：负责整个Netty的启动、初始化、服务器连接等过程，主要负责客户端的引导，可以连接远程服务器，只绑定一个EventLoopGroup
  - ServerBootStrap：负责服务端的引导。用于服务端启动绑定本地端口，绑定两个EventLoopGroup。这两个被称为Boss和Worker。
    - Boss负责监听网络连接事件，新的连接到达，将Channel注册到Worker
    - Worker分配一个EventLoop处理Channel的读写实现，通过Selector进行事件循环
  - Channel：网络通信的载体，提供了较NIO的Channel更高层次的抽象。会有生命周期，每一种状态都会绑定事件回调。
- 事件调度层：通过Reactor线程模型对各类事件进行处理，通过Selector主循环线程集成多种事件，实际的处理逻辑交给服务编排层的Handler完成。
  - - 
  - **EventLoop**：本质上是一个单线程执行器，内部维护一个Selector，run方法处理Channel在生命周期内的所有IO事件，比如accept、connect、read、write等。同一时间与一个线程绑定，负责处理多个Channel。
    - 继承关系：
      - 继承netty的OrderedEventExecutor。
        - 拥有netty特有的自定义线程池方法：包括判断一个线程是否属于当前EventLoop。
      - 继承JUC的ScheduledExecutorService，因此其含有线程池的方法
  - **EventLoopGroup**：本质是一个线程池，负责接收IO请求，并分配线程去执行。内部有EventLoop。是Reactor线程模型的具体实现方式，channel调用group的register方法来绑定相应的EventLoop，通过传入不同参数，支持Reactor的三种线程模型：
    - 单线程模型：EventLoopGroup内部包含一个EventLoop，Boss和Worker同时使用一个EventLoopGroup。
    - 多线程模型：EventLoopGroup内部包含多个EventLoop，Boss和Worker同时使用一个EventLoopGroup。
    - 主从多线程模型：EventLoopGroup内部包含多个EventLoop，Boss是主Reactor，Worker是从Reactor，使用不同的EventLoopGroup，主Reactor负责新连接的Channel创建，然后将Channel注册到从Reactor。
- 服务编排层：负责组装各类服务，实现对网络事件的处理
  - **ChannelPipeline**：负责组装各种ChannelHandler，内部通过双向链表来管理ChannelHandler，IO事件触发时，依次调用ChannelHandler来对数据进行拦截和处理。一个 ChannelPipeline 关联一个 EventLoop。
    - 客户端和服务端都有各自的ChannelPipeline，数据从客户端发向服务端，该过程称为出站。反之为入站。
  - **ChannelHandler**：负责数据的编解码和加工处理，分为InBound和OutBound，分别负责入站和出站。每一个ChannelHandler都会绑定一个ChannelHandlerContext。
    - ChannelInboundHandler：入站处理器的父类，用于读取客户端数据，写回结果。
    - ChannelOutboundHandler：出站处理器的父类，用于处理写回结果。
  - **ChannelHandlerContext**：用于保存ChannelHandler的上下文，通过它可以知道handler和pipeline的关联关系，可以实现handler之间的交互。它包含了 ChannelHandler 生命周期的所有事件，如 connect、bind、read、flush、write、close 等。

![Netty-framework](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/Netty-framework.png)


---

> Author:   
> URL: http://localhost:1313/posts/netty/netty%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%B8%80-%E6%A6%82%E8%A7%88/  

