# 编译OpenJDK


# 前言

最近在看《深入理解Java虚拟机》这本书，1.6节是编译openjdk，自己也想尝试一下。

使用操作系统为：Ubuntu 18.04 LTS

# 正文



## 下载源码

&gt; http://hg.openjdk.java.net/jdk/jdk12

书中提供的下载地址由于网络问题下载较慢，所以在Github的仓库下载，地址为：
&gt; https://github.com/openjdk/jdk

作者建议阅读**doc/building.md**，里面有详细的需要安装的编译环境与编译说明。



## 构建编译

使用**bash configure**进行设置编译参数。具体的编译配置参数可见**doc/building.md**。
成功后命令行回显如下：

![编译成功回显](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/buildjdk/bashconfigure.png)



## 编译

输入**make images**进行编译。等大概二十分钟编译完成。

编译好的jdk位于**build/配置名称/jdk**，该目录可以直接当成完整的JDK进行使用。

至此，编译JDK工作完成。

---

> Author:   
> URL: http://localhost:1313/posts/%E5%B7%A5%E5%85%B7/build-jdk/  

