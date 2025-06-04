# Bitcask论文阅读笔记




## 简介

Bitcask是Riak分布式数据库使用的日志型存储模型，主要有以下几点特性：

- 读写低延迟
- 高吞吐量，尤其是随机写
- 能处理大量的数据集而不降低性能
- 故障时快速恢复且不丢失数据
- 易于备份和恢复
- 易于理解的代码结构和数据格式



## 结构



### 硬盘结构

在给定的一段时间内，只有一个`active data file`能提供`append`（写入）功能，当该文件大小达到指定阈值时，则会创建一个新的`active data file`。每一个写入到file的entry结构如下：



![Entry结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/distribute%20system/bitcask/entry.jpg)



包括以下内容：

- crc：校验值
- tstamp：记录写入该数据的时间
- ksz：key_size
- value_sz：value_size
- key：key
- value：value



### 内存结构

在写入之后，一个被称为`keydir`（hash table）的内存结构将被更新，每一个键映射到固定大小的结构中，包括了以下内容：

- file_id：去哪个文件查询
- value_size：读取value多长的字节
- value_position：value在文件中的位置

每一次写操作发生，`keydir`都会进行原子性的更新。





## 操作



### read

读取value的过程：

1. 查找keydir中的key，并读取`file_id`，`position`，`size`。
2. 通过`file_id`找到对应的`data file`，再通过`position`和`size`字段找到entry中的value。



### merge

由于删除操作并不会真正删除掉entry，只是将删除操作封装成entry写入文件，因此需要`merge`操作将所有的非`active`文件中entry遍历，并重组为一个新文件。同时生成一个`hint file`，用于存储`data file`的位置和大小。



&gt; Q：hint file的作用是什么？

A：每次进程重启时需要重建`keydir`，需要扫描所有的数据文件，因此使用`hint file `加速构建`keydir`的速度。





---

> Author:   
> URL: http://localhost:1313/posts/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/bitcask/  

