# FastDFS使用入门


&lt;!-- more --&gt;

## 介绍

FastDFS是一个开源的分布式文件系统，主要有以下功能。

分布式文件系统（Distributed File System）是指文件系统管理的物理存储资源不一定直接连接在本地节点上，而是通过计算机网络与节点相连。 

FastDFS主要有以下特点：

- 文件存储
- 文件同步
- 文件访问（上传、下载）
- 存取负载均衡
- 在线扩容





## FastDFS的架构

![1526205318630.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/1526205318630.png)

由三个部分组成：Client，Tracker Server，Storage Server。

Client是上传下载数据的服务器，Tracker Server是跟踪服务器，主要做调度工作，起到负载均衡，管理所有的Storage Server。Storage Server：存储服务器，主要提供容量和备份服务。





## FastDFS安装



### 安装环境

操作系统：Ubuntu 18.04



#### 下载libfastcommon

```bash
wget https://github.com/happyfish100/libfastcommon/archive/V1.0.7.tar.gz
```



#### 解压

```bash
tar -zxvf V1.0.7.tar.gz
cd libfastcommon-1.0.7
```



#### 编译 安装

```bash
./make.sh
./make.sh install
```



#### 设置软链接

```bash
ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so
```



#### 安装FastDFS

```bash
wget https://github.com/happyfish100/fastdfs/archive/V5.05.tar.gz
tar -zxvf V5.05.tar.gz
cd fastdfs-5.05
./make.sh
./make.sh install
```



#### **配置Tracker服务器**

```bash
cd /etc/fdfs
cp tracker.conf.sample tracker.conf
vim tracker.conf
```

修改`base_path`为指定目录



#### 启动Tracker服务器

```bash
fdfs_trackerd /etc/fdfs/tracker.conf start
```



#### 关闭Tracker服务器

```bash
fdfs_trackerd /etc/fdfs/tracker.conf stop
```



启动后查看是否运行成功

```bash
netstat -unltp|grep fdfs

```





#### 配置Storage服务器

```bash
cd /etc/fdfs
cp storage.conf.sample storage.conf
vim storage.conf

```



修改 `base_path`，`store_path0`，`tracker_server`三个配置项



#### 启动Storage服务器

```bash
fdfs_storaged /etc/fdfs/storage.conf start

```



查看存储节点的集群状态信息

```bash
/usr/bin/fdfs_monitor /etc/fdfs/storage.conf

```



### 测试文件上传

```bash
/usr/bin/fdfs_upload_file client.conf /tmp/1.txt

```



---

> Author:   
> URL: http://localhost:1313/posts/linux/fastdfs-01/  

