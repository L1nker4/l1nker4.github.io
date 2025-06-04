# 分析Linux中的Zero-Copy技术




## 传统IO的流程

零拷贝技术是对传统IO的性能优化，在介绍零拷贝之前，先简单了解一下传统IO中，数据流向的过程：

1. 用户进程发起`read()`调用，从用户态切换至内核态，DMA从文件中读取数据，并存储在IO Buffer（内核态）
2. 请求得到的数据从IO Buffer拷贝到用户态Buffer（从内核态切换到用户态），然后返回给用户进程。
3. 用户进程调用`write()`将数据输出到网卡中，此时将从用户态切换到内核态，请求数据从用户进程空间拷贝到Socket Buffer。
4. `write()`调用结束后返回用户进程，此时从内核态切换到用户态。

整个过程中涉及4次上下文切换、4次数据拷贝（2次CPU拷贝 &#43; 2次DMA拷贝）。



我们需要对该流程进行优化，以提高IO的性能，因此诞生了零拷贝的概念。



## 零拷贝



&gt; &#34;**Zero-copy**&#34; describes [computer](https://en.wikipedia.org/wiki/Computer) operations in which the [CPU](https://en.wikipedia.org/wiki/Central_processing_unit) does not perform the task of copying data from one [memory](https://en.wikipedia.org/wiki/RAM) area to another or in which unnecessary data copies are avoided. This is frequently used to save CPU cycles and memory bandwidth in many time consuming tasks, such as when transmitting a [file](https://en.wikipedia.org/wiki/Computer_file) at high speed over a [network](https://en.wikipedia.org/wiki/Telecommunications_network), etc., thus improving [performances](https://en.wikipedia.org/wiki/Computer_performance) of [programs](https://en.wikipedia.org/wiki/Computer_program) ([processes](https://en.wikipedia.org/wiki/Process_(computing))) executed by a computer.



维基百科对于零拷贝的定义为：解放CPU对数据的拷贝操作，以提高IO操作的性能。



### 零拷贝解决的问题

- 减少了内核和用户空间的数据拷贝操作，即上下文切换的开销。
- 减少了内核缓冲区的数据拷贝。
- 用户进程直接访问硬件接口，而避免访问内核空间。



### 实现方式

零拷贝是一个概念，那么则会存在多种实现方式，实现上整体可以分为以下三大类：

- **减少内核空间和用户空间之间的数据拷贝**
  - mmap()： 将内核Buffer和用户Buffer进行映射，实现两者之间的共享，减免了之间的拷贝。
  - sendfile()：内核态中实现将文件发送到Socket。
  - sendfile() with DMA：Linux 2.4中对`sendfile()`的优化，通过DMA减少了拷贝次数。
  - splice()：实现了不通过内核与用户态之间的拷贝操作完成两个文件的拷贝。。
  - send() with MSG_ZEROCOPY：Linux 4.14中提供，用户进程能把用户空间的数据通过零拷贝的方式经过内核空间发送到Socket
- **允许用户态进程直接和硬件进行数据传输**
  - 用户态进程直接访问硬件：理论上最高效的技术，存在严重的安全问题，并不是最优方案。
  - 内核控制访问硬件：比上一种方案更加安全，存在问题：DMA传输过程中，用户缓冲区内存页被锁定以保证数据一致性
- **IO Buffer（内核）和用户Buffer的传输优化**
  - Copy-on-Write：多个进程共享一块数据时，读操作的进程直接访问内核空间的数据，需要执行修改操作才将数据拷贝到自己的用户空间。
  - Buffer Sharing：每个进程维护一个缓冲区池，这个区域能被同时映射到用户空间和内核空间，内核和用户共享这个区域，这样避免了拷贝操作。



---

> Author:   
> URL: http://localhost:1313/posts/linux/%E5%88%86%E6%9E%90zero-copy%E6%8A%80%E6%9C%AF/  

