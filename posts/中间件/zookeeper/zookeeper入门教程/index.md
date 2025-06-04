# Zookeeper入门教程




## 简介

Zookeeper为分布式系统提供了高效可靠的分布式协调服务，其本质是一个键值存储系统，提供了诸如命名服务、配置管理、分布式锁等服务。其采用ZAB协议对集群数据的一致性进行管理。

它负责存储和管理一些数据，然后接受观察者的注册，一旦数据状态发生变化，Zookeeper负责通知观察者做出相应的反应。



几个特点：

- 一个Leader，多个Follow组成的集群。
- 半数以上节点存活，集群即可正常服务，适合部署奇数台节点。
- 全局数据一致，每个节点都保存相同的数据副本。
- 所有客户端看到的数据都是一致的，并且请求按照顺序执行（FIFO）
- 数据更新原子性。
- 更新删除操作都是基于事务的，是用于**读多写少**环境。



### 数据模型

Zookeeper的数据模型由一个树形结构构成，每个节点称为一个ZNode，由于设计目标是实现协调服务，而不是数据存储，故默认存储大小为1MB，每个ZNode可以通过其路径唯一标识。







## 运行

从官网下载Zookeeper的二进制发布版，解压后得到以下文件：

```
zero@Pluto:~/Zookeeper/apache-zookeeper-3.5.7-bin$ ls
LICENSE.txt  NOTICE.txt  README.md  README_packaging.txt  bin  conf  docs  lib
```



执行`bin/zkServer.sh version`，看到版本信息说明正常运行。

```
zero@Pluto:~/Zookeeper/apache-zookeeper-3.5.7-bin$ bin/zkServer.sh version
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /home/zero/Zookeeper/apache-zookeeper-3.5.7-bin/bin/../conf/zoo.cfg
Usage: bin/zkServer.sh [--config &lt;conf-dir&gt;] {start|start-foreground|stop|restart|status|print-cmd}
```



### 单机部署

创建一个zoo.cfg文件，内容如下：

```
tickTime=2000

initLimit=10

syncLimit=5

dataDir=/opt/Zookeeper-3.5.7/zkData

clientPort=2181
```



执行`bin/zkServer.sh start`启动服务器节点。

```
zero@Pluto:/opt/Zookeeper-3.5.7$ bin/zkServer.sh start
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/Zookeeper-3.5.7/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

zero@Pluto:/opt/Zookeeper-3.5.7$ jps -l
3892 sun.tools.jps.Jps
3850 org.apache.zookeeper.server.quorum.QuorumPeerMain
```



执行`bin/zkCli.sh`启动客户端。

```
zero@Pluto:/opt/Zookeeper-3.5.7$ bin/zkCli.sh
/usr/bin/java
Connecting to localhost:2181
2022-04-29 17:08:00,703 [myid:] - INFO  [main:Environment@109] - Client environment:zookeeper.version=3.5.7-f0fdd52973d373ffd9c86b81d99842dc2c7f660e, built on 02/10/2020 11:30 GMT
2022-04-29 17:08:00,705 [myid:] - INFO  [main:Environment@109] - Client environment:host.name=Pluto.localdomain
2022-04-29 17:08:00,705 [myid:] - INFO  [main:Environment@109] - Client environment:java.version=1.8.0_312
2022-04-29 17:08:00,707 [myid:] - INFO  [main:Environment@109] - Client environment:java.vendor=Private Build
2022-04-29 17:08:00,707 [myid:] - INFO  [main:Environment@109] - Client environment:java.home=/usr/lib/jvm/java-8-openjdk-amd64/jre
```



执行`bin/zkServer.sh stop`停止服务端。

### 配置参数

配置参数如下：

- tickTime=2000：通信心跳时间，作为Zookeeper的基本时间单位。单位为ms。
- initLimit=10：leader和follower的初始连接能容忍的最大心跳数量（tickTime的数量）。
- syncLimit=5：leader和follower同步通信时限，超过指定时间，leader认为follower宕机，从列表中删除follower。
- dataDir：数据存储路径，包括快照和事务日志。
- clientPort=2181：客户端连接端口。



## 基础理论



### 节点信息



- cZxid：创建节点的事务zxid
  - zxid：每次修改Zookeeper状态都会产生一个事务id，每次修改都有唯一的zxid。
- ctime：znode被创建的毫秒数
- mZxid：znode最后更新的事务zxid
- mtime：znode最后修改的毫秒数
- pZxid：znode最后更新的毫秒数zxid
- cversion：znode节点修改次数
- dataVersion：znode数据修改次数
- aclVersion：访问控制列表修改次数
- ephemeralOwner：如果是临时节点，这个是znode拥有者的session id，不是临时则为0
- dataLength：znode的数据长度
- numChildren：znode子节点数量



Znode有以下两种类型：

- 持久（**PERSISTENT** ）：客户端与服务端断开连接后，节点不删除。除非主动执行删除操作。
- 临时（**EPHEMERAL** ）：客户端断开连接后，创建的节点被删除。



### 集群角色

Zookeeper集群中各个服务器都会承担一种角色，分别有以下三种：

- **Leader**：领导者节点，只有一个。
  - 会给其他节点发送心跳信息，通知其在线。
  - 所有写操作都通过Leader完成再由ZAB协议将操作广播给其他服务器。
- **Follower**：跟随者节点，可以有多个。
  - 响应Leader的心跳。
  - 可以直接响应客户端的读请求。
  - 将写请求转发给Leader进行处理。
  - 负责在Leader处理写请求时进行投票。
- **Observer**：观察者，无投票权。





### ACL

Zookeeper采用ACL（Access Control Lists）机制来进行权限控制，每个Znode创建时都会有一个ACL列表，由此决定谁能进行何种操作。定义了以下五种权限：

- **CREATE：**允许创建子节点；
- **READ：**允许从节点获取数据并列出其子节点；
- **WRITE：** 允许为节点设置数据；
- **DELETE：**允许删除子节点；
- **ADMIN：** 允许为节点设置权限。





### 读写流程

- 所有节点都可以直接处理读操作，从本地内存中读取数据并返回给客户端。

- 所有的写操作都会交给Leader进行处理，其余节点接收到写请求会转发给Leader，基本步骤如下：
  1. Leader为每个Follower分配了一个FIFO队列，Leader接收到请求，将请求封装成事务存放于队列中发送给Follower并等待ACK。
  2. Follower接收到事务请求后返回ACK。
  3. Leader收到超过半数的ACK后，向所有Follower和Observer发送Commit完成数据提交。
  4. Leader将处理结果返回给客户端。



### 事务

为了实现**顺序一致性**，Zookeeper使用递增的事务id（zxid）来标识事务。zxid是一个64位的数字，高32位是`epoch `，该部分记录Leader是否改变（每一次选举都会有一个新的epoch），低32位用于递增计数。



### watch机制

客户端可以注册监听znode状态，当其状态发生变化，监听器被触发，并主动向客户端推送相应的消息。



### 会话

几个要点：

- 客户端通过**TCP长连接**连接到Zookeeper集群。
- 第一次连接开始就建立，之后通过**心跳检测机制**来保持有效的会话状态。
- 客户端中配置了集群列表，客户端启动时遍历列表尝试连接，失败则尝试连接下一个。
- 每个会话都有一个**超时时间**，**如果服务器在超时时间内没有收到任何请求，则会话被认为过期**，并将与该会话相关的临时znode删除。因此客户端通过心跳（ping）来保持会话不过期。
- Zookeeper的会话管理通过`SessionTracker `完成，采用了**分桶策略**进行管理，以便对同类型的会话统一处理。
  - 分桶策略：将类似的会话放在同一区块中进行管理。



会话的四个属性：

- **sessionID**：会话的全局唯一标识，每次创建新的会话，都会分配新的ID。
- **TimeOut**：超时时间，客户端会配置`sessionTimeout，连接Zookeeper时会将此参数发送给Zookeeper服务端，服务端根据自己的超时时间确定最终的TimeOut。
- **TickTime**：下次会话超时时间点，便于Zookeeper检查超时会话。
- **isClosing**：标记会话是否被关闭。







## ZAB协议

ZAB协议与Paxos算法类似，是一种保证数据一致性的算法，ZAB协议是ZK的数据一致性和高可用的解决方案。因此ZAB协议主要完成两个任务：

- **实现一致性**：Leader通过ZAB协议给其他节点发送消息来实现**主从同步**，从而保证数据一致性。
- **Leader选举**：当ZK集群刚启动或Leader宕机，通过选举机制选出新的Leader，从而保证高可用。



Zookeeper集群使用**myid**来标识节点的唯一ID，`myid`存放于ZK数据目录下的`myid`文件。



服务器存在四种状态，分别如下所述：

- **LOOKING**：不确定Leader状态，节点认为集群中没有Leader，会发起Leader选举。
- **FOLLOWING**：跟随者状态，节点为Follower，知道Leader是哪个节点。
- **LEADING**：领导者状态，节点为Leader，并且会维护与Follower之间的心跳。
- **OBSERVING**：观察者状态，节点为Observer，不参加选举和投票。



### 选举

每个服务器启动都会判断当前是否进行选举，若在恢复模式下，刚从崩溃状态恢复的服务器会从磁盘快照中恢复数据。

每个服务器进行选举时，会发送以下的信息：

- **logicClock**：每个节点维护一个自增的整数，表明是该服务器进行的第多少轮投票。
- **state**：服务器状态。
- **self_id**：服务器的`myid`。
- **self_zxid**：当前服务器上保存数据的最大`zxid`。
- **vote_id**：投票对象的服务器`myid`。
- **vote_zxid**：投票对象的最大`zxid`。



**投票流程**：

1. 每个服务器在开始新一轮投票时，都会对自己的`logicClock`进行自增操作。
2. 每个服务器清空投票箱中上一轮的投票数据。
3. 每个服务器最开始都投票给自己，并广播。
4. 服务器接收其他服务器的投票信息并计入投票箱。
5. 判断投票信息中的`logicClock `，并做不同处理。
   - 若外部投票`logicClock ` &gt; 本节点的`logicClock `，说明本节点轮次小于其他节点，立即清空投票箱，并更新`logicClock `为最新，并重新投票并广播。
   - 若外部投票`logicClock ` &lt; 本节点的`logicClock `，忽略此投票信息，处理下一个。
   - 若外部投票`logicClock ` = 本节点的`logicClock `，比较`vote_zxid`，若小于外部节点，则将`vote_id`改为外部投票中的`vote_id`并广播。
6. 如果确定有过半服务器给自己投票，则终止投票，否则继续接收投票信息。
7. 投票终止后，若服务器赢得投票，将状态改为`LEADING`，否则改为`FOLLOWING`。



### 数据一致性

ZAB保证数据一致性的流程可以说是二阶段提交的简化版，其更新逻辑如下：

- 所有写请求都要转发给Leader，Leader使用原子广播将事务通知给Follower。
- Follower接收到信息后以**事务日志**的形式将数据写入，并返回ACK响应给Leader。
- 当**半数以上**的Follower更新完成后，Leader对更新进行Commit，然后给客户端返回一个更新成功的响应。



ZAB对**Leader发生故障**而导致数据不一致问题做了相应的措施：

- 已经在Leader上提交的事务，其他节点已经持久化，不做处理。
- 只在Leader上创建而Follower未ACK，Leader需要回滚。





## 应用场景

- **统一命名服务**：通过顺序节点特性可以生成**全局唯一ID**，从而提供命名服务。
- **统一配置管理**
  - 一般集群中，所有配置信息是一致的，例如Kafka集群。
  - 对配置文件修改后，需要快速同步到各个节点上。
- **分布式锁**：使用临时节点和Watch机制实现。举例说明：
  - 分布式系统中有三个节点：A，B，C，通过访问ZK实现分布式锁功能。
  - 所有节点访问`/lock`，并创建**带序号的临时节点**。
  - 节点尝试获取锁，并获取`/lock`下所有子节点，判断自己创建的节点是否为最小。
    - 是，认为拿到锁。
    - 否，未获取到锁，监听节点变化，若其他线程释放，则可能拿到锁。
  - 释放锁时，将创建的节点删除。
- **统一集群管理**
  - 实时掌握每个节点的状态，并且做出一些相应的调整。
  - 动态监听分布式系统中服务的上下线，从而实现动态扩容。
  - 提供集群选主功能：让所有服务节点创建同一个Znode，由于路径唯一，因此只能有一个节点能创建成功，创建成功的为分布式系统中的Leader。







---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/zookeeper/zookeeper%E5%85%A5%E9%97%A8%E6%95%99%E7%A8%8B/  

