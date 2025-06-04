# Kafka学习笔记(三)-通信协议




## 协议设计



需要进行网络传输的中间件都会拥有自己的一套通信协议，这往往会成为该组件的性能瓶颈，需要考虑的优化点较多。Kafka自定义了基于TCP的二进制通信协议，Kafka2.0中，一共有43种协议类型，每个都有对应的请求和响应，与HTTP协议类似，它同样有`RequestHeader`和`RequestBody`。其中`RequestHeader`结构如下：

- api_key：API标识，例如PRODUCE、FETCH等，用于分别请求的作用。
- api_version：API版本号
- correlation_id：客户端指定的唯一标识，服务端返回响应需要将该字段返回以此对应。
- client_id：客户端id



Kafka除了提供基本数据类型，还提供了以下的特有类型：

- nullable_string：可为空的字符串类型，若为空用-1表示
- bytes：表示字节序列，开头是数据长度N（int32表示），后面是N个字节
- nullable_bytes：与上述string相同
- records：表示Kafka中的消息序列
- array：表示一个给定类型T的数组



`RequestBody`结构如下：

- transactional_id：事务id，不使用事务，此项为null
- acks：对应客户端的acks参数
- timeout：超时时间
- topic_data：要发送的数据集合，array类型
  - topic：主题
  - data：数据，array类型
    - partition：分区编号
    - record_set：数据



`Response`结构如下：

- ResponseHeader
  - correlation_id：与请求相对应
- ResponseBody
  - responses：array类型，返回的响应结果
    - topic：主题
    - partition_responses：返回结果，array类型
      - partition：分区编号
      - error_code：错误码，用来标识错误类型
      - base_offset：消息集的起始偏移量
      - log_append_time：消息写入broker的时间
      - log_start_offset：所在分区起始偏移量


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%B8%89-%E9%80%9A%E4%BF%A1%E5%8D%8F%E8%AE%AE/  

