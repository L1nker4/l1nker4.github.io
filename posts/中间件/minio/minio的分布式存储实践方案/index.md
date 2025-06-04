# MinIO的分布式存储实践方案




## 简介

MinIO是一个开源的分布式对象存储组件，它兼容Amazon S3 API，适合于存储大容量的非结构化数据，支持单个对象最大5TB。



MinIO特点：

- 部署简单，仅需要单独一个二进制文件
- 支持纠删码机制，能恢复部分数据块丢失的情况。
- 读写性能高

![MinIO Benchmark](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20220924181123151.png)

## 基础原理

### 纠删码

纠删码是分布式存储领域常见的一种冗余技术，与副本机制相对比，纠删码拥有更高的磁盘利用率。纠删码的基本原理：通过**纠删码算法**对原始数据进行计算，得到冗余的编码数据，并将数据和冗余编码一起存储，如果未来存储介质发生故障，导致其中部分数据出错，此时可以通过对应的重构算法，**解码**出完整的原始数据，以达到容错的目的。即**总数据块 = 原始块 &#43; 校验快**($n = k &#43; m$)。纠删码技术的磁盘利用率为$k / (k &#43; m)$，允许总数据块中任意m个数据块损坏。

上面提到的n、m的比值，是衡量纠删码的核心参数，这个值被称为冗余度，冗余度越高（校验快越多），允许丢失的数据块可以越多，同时数据存储成本也就越高。k值决定数据分块的粒度，k越小，数据分散度越小、重建代价越大。k值越大，数据拷贝的负载越大。常见的公有云独享存储的冗余度一般在`1.2-1.4`左右。

目前常用的纠删码算法：`Reed-Solomon`，它有两个参数n和m，记为$RS(n , m)$。n代表原始数据块个数。m代表校验块个数。



下图中是使用16块磁盘作为存储设备的情况，假设此时MinIOn持有16个磁盘，MinIO会将其中8块作为数据盘，另外八块作为校验盘，数据盘存储对象的原始数据，校验盘存储对象的校验数据。纠删码默认配置是**1:1**，也就是将所有磁盘中的一半作为数据盘，一半做为校验盘。同时MinIO使用HighwayHash编码计算数据块的hash值，获取文件时会计算hash值来校验文件的准确性。

![纠删码的磁盘布局](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/erasure-code1.jpg)

- 纠删码缺点
  - 需要读取其他的数据块和校验块
  - 编码解码需要消耗CPU资源
- 纠删码优点
  - 副本机制对于大文件机极其消耗磁盘空间，纠删码可以通过较少的磁盘冗余，较为高效的解决数据丢失的问题。
- 应用场景
  - 对于不被长期访问的冷数据，采用纠删码技术，可以大大减少副本数量。



### Server Pool

使用minio server指令创建的MinIO节点集合，提供对象存储和处理请求的功能。

MinIO可以通过增加Server Pool的方式，实现集群的横向扩展。

当有新的Server Pool加入Cluster，存储的元数据会进行同步，但是其他Server Pool已存储对象不会同步。



举例：

- minio server https://minio{1...4}.example.net/mnt/disk{1...4}代表一个Server Pool，其中有四个server节点各有4块磁盘。

- minio server https://minio{1...4}.example.net/mnt/disk{1...4} https://minio{5...8}.example.net/mnt/disk{1...4}代表有两个Server Pool。



MinIO选择Server Pool策略；选择剩余空间最大的Server Pool进行存储。



![MinIO选择策略](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20221113153730493.png)



### 存储级别

MinIO目前支持两种存储级别：Reduced Redundancy和Standard，提供两种不同的级别来修改数据块和校验块的比例。MinIO使用**EC:N**来表示EC Set中存储校验块的磁盘数量，N越大，容错能力越强，但占用磁盘空间越多。

可以通过在S3 Put API中添加x-amz-storage-class参数来指定当前文件的存储级别。



- Standard：默认使用的存储级别，EC:N参数与Set中磁盘数量有关。可通过环境变量MINIO_STORAGE_CLASS_STANDARD=EC:N来设置，N不能大于磁盘数量的一半。
- Reduced Redundancy：使用比Standard级别更少的磁盘数量存储校验块。通过环境变量MINIO_STORAGE_CLASS_RRS=EC:N来设置。默认参数为EC:2

![存储级别设置](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20221113154043570.png)

### Erasure Sets

MinIO集群会将所持有的磁盘设备（数据盘&#43;校验盘）划分为多个**Erasure Sets（EC Set）**。EC Set的磁盘数量取值为**4-16之间**。Ec Set具体划分情况参考2.1节表格。文件通过S3 API传入集群后，通过分片算法找到**具体的存储Set**，假设当前Set拥有16个磁盘，按照默认的纠删策略，会将整个文件划分成8个数据分片(data shards)，再通过纠删码算法算出8个校验分片（parity shards）。

![MinIO EC Set模型](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/minio-architecture.png)



MinIO使用**EC:N**来表示**EC Set**中存储校验块的磁盘数量，N越大，容错能力越强，占用磁盘空间越多。假设使用EC:2策略，当前总共有4个磁盘，对象会被分为2个数据块，2个奇偶校验块，共4个块，其中允许任意2个出错，但是能正常**读**数据。但是如果需要保证正常**读写，**需要保证3个磁盘正常（N&#43;1）。上面的参数N可通过**MINIO_STORAGE_CLASS_STANDARD**参数来指定，最大值为磁盘总数量的一半，最小值为2（开启纠删码功能最小磁盘数量为4）。



EC Set的磁盘数量和对应的默认纠删策略对应如下：

![image-20220819174124842](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20220819174124842.png)



### 读写流程



- **Write**：根据对象名计算出hash得到所处的EC Set，然后创建临时目录，将对象数据写入，直到所有数据写入完成，接下来每次读取10MB，进行EC编码，并将编码数据保存，然后写入meta信息，最终将数据移动到指定存储位置，并删除临时目录。

- **Read**：根据对象名获取对应的EC Set，然后去读取meta信息，通过meta信息去做解码，每次读取10MB，在纠删码特性中，如果是默认1:1策略，只需要读取N/2的数据即可完成解码。





- Server Pool：通过minio server指定的服务端节点集合，可以提供对象存储和检索请求等功能。当有新的Server Pool加入Cluster，bucket的元数据会进行同步，但是其他Server Pool已存储的对象不会同步到新的Server Pool。
  - 例如minio server https://minio{1...4}.example.net/mnt/disk{1...4}  代表一个Server Pool，其中有四个server节点各有4块磁盘。
- Cluster：集群中包括一个或多个Server Pool。每个hostname参数代表一个Server Pool。MinIO均匀地将对象放入更为空闲的Server Pool。
  - minio server https://minio{1...4}.example.net/mnt/disk{1...4}https://minio{5...8}.example.net/mnt/disk{1...4} 代表有两个Server Pool。





## 部署方案



![部署方案图](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/minio-server-startup-mode.png)



### 单机部署



单机单磁盘：

```
./minio server --address :50851 --console-address :19003 /data01/miniotest
```



单机多磁盘（最低四个盘开启纠删功能）

```
./minio server --address :50851 --console-address :19003 /data01/miniotest /data02/miniotest /data03/miniotest /data04/miniotest

./minio server --address :50851 --console-address :19003 /data0{1...4}/miniotest
```





| 挂载磁盘数量（n） |    EC  Set情况    |
| :---------------: | :---------------: |
|   4 &lt;=  n &lt;=16    |        1个        |
|        18         | 2个，每个9个磁盘  |
|        19         |       报错        |
|        20         | 2个，每个10个磁盘 |
|        31         |       报错        |
|        32         | 2个，每个16个磁盘 |
|        33         | 3个，每个11个磁盘 |
|        34         |       报错        |
|        35         | 5个，每个7个磁盘  |
|        36         | 3个，每个12个磁盘 |

经过验证，挂载数量4 &lt;= n &lt;=16时，EC Set只有一个，并且set中磁盘数即为n。16 &lt;= n &lt;= 32时，n只能取被2整除。

如果n &gt; 32时，n必须有以下集合的公因数（用a表示），共有n / a 个EC Set，每个set有a个磁盘。

集合为：[4 5 6 7 8 9 10 11 12 13 14 15 16]



### 集群部署

由于集群需要保证数据一致性，因此MinIO使用分布式锁[Dsync](https://github.com/minio/dsync)来实现，Dsync在节点数量少的时候性能很高，随着节点数量增加，性能会出现下降趋势，能保证的最大节点数量为32。如果当前集群节点数量已到达32个，但是仍需要扩容，可以考虑**多集群方案**：通过负载均衡组件，可以根据header、文件名hash等将请求转发给不同的集群。

分布式部署需要有相同的账号密码，集群节点的时间差不能超过3秒，需要使用NTP来保证时间的一致。集群节点数量必须是大于等于4的偶数。官方建议使用Nginx、HAProxy等组件来实现负载均衡。受限于纠删码策略，MinIO扩容的机器数量必须保持和原有集群数量大小相同或为倍数。不支持扩容单个节点等操作。



将n01-n03三台主机下面的data01-data04的miniotest1作为存储位置。共12块磁盘

```
./minio server --address :50851 --console-address :19003 http://n0{1...3}.bda.test.com/data0{1...4}/miniotest1
```



扩容时，末尾添加扩容的节点，并重启集群：

```
./minio server --address :50851 --console-address :19003 http://n0{1...3}.bda.test.com/data0{1...4}/miniotest1 http://n0{1...3}.bda.test.com/data0{5...8}/miniotest1
```



## 监控方案



设置环境变量`MINIO_PROMETHEUS_AUTH_TYPE=&#34;public&#34;`，并在`Prometheus.yml`添加如下配置：

```yml
scrape_configs:
- job_name: minio-job
  metrics_path: /minio/v2/metrics/cluster
  scheme: https
  static_configs:
  - targets: [&#39;minio.example.net:9000&#39;]
```


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/minio/minio%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E5%AE%9E%E8%B7%B5%E6%96%B9%E6%A1%88/  

