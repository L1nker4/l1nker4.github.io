# Docker学习笔记


## 1.Docker概念



### 模型

![docker.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/docker/docker.png)

- 虚拟机运行在虚拟的硬件上，应用运行在虚拟机内核上，而Docker是机器上的一个进程，Docker应用是Docker的一个子进程。
- Docker是对Linux容器（LXC）的一种封装，提供简单易用的接口，Docker是目前最流行的Linux容器解决方案。



### Docker用途

- 提供一次性的环境。主要用于测试。
- 提供云服务
- 组建微服务架构。



### Docker引擎

Docker 引擎是一个包含以下主要组件的客户端服务器应用程序。

- 一种服务器，它是一种称为守护进程并且长时间运行的程序。
- REST API用于指定程序可以用来与守护进程通信的接口，并指示它做什么。
- 一个有命令行界面工具的客户端。

![620140640_31678.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/docker/620140640_31678.png)

### Docker系统架构

Docker 使用客户端-服务器 (C/S) 架构模式，使用远程 API 来管理和创建 Docker 容器。

Docker 容器通过 Docker 镜像来创建。

容器和对象的关系可以类似于对象和类。

![262150629_86976.png](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/docker/262150629_86976.png)





### Docker镜像

在Linux下，内核启动会挂载`root`文件系统，Docker镜像就相当于一个`root`文件系统，它除了提供容器运行时所需的程序，资源，配置之外，还有一些配置参数（匿名卷，环境变量，用户）。

Docker镜像使用分层存储技术，它并非像ISO镜像那样的文件，镜像是一个虚拟的概念，它是由一组文件系统构成。



### Docker容器

容器的本质是进程，容器进程运行于属于自己的命名空间，它可以拥有自己的`root`文件系统，自己的网络配置等宿主机可以有的东西。

每一个容器运行时，都是以镜像为基础层，在其上创建一个当前容器的存储层，可以称这个为容器运行时读写而准备的存储层为**容器存储层**。



### Docker仓库

存放镜像的地方



## 2.安装Docker



#### 1.获取脚本并下载

```
//从官网获取安装脚本
$ curl -fsSL get.docker.com -o get-docker.sh
//使用AzureChinaCloud镜像脚本
$ sudo sh get-docker.sh --mirror AzureChinaCloud
```



#### 2.启动Docker

```
$ sudo systemctl enable docker
$ sudo systemctl start docker
```



#### 3.建立Docker用户组

docker命令只有root用户和docker组的用户才能访问docker引擎



##### 建立Docker组

```
$ sudo groupadd docker
```



##### 将当前用户加入Docker组

```
$ sudo usermod -aG docker $USER
```



##### 校验是否安装成功

```shell
l1nker4@zero:~$ docker version 
Client:
 Version:           18.09.6
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        481bc77
 Built:             Sat May  4 02:35:57 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.6
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.8
  Git commit:       481bc77
  Built:            Sat May  4 01:59:36 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```



#### 4. 添加镜像加速器



##### 新建daemon.json文件

```shell
$ sudo touch /etc/docker/daemon.json
```

##### 添加内容

```json
{
  &#34;registry-mirrors&#34;: [
    &#34;https://registry.docker-cn.com&#34;
  ]
}
```



## 3.常用命令



#### 1.镜像操作

| 名称 |        命令        |      作用      |
| :--: | :----------------: | :------------: |
| 搜索 | docker search xxx  |    搜索镜像    |
| 拉取 |  docker pull xxx   | 从仓库拉取镜像 |
| 列表 |   docker images    |  列出所有镜像  |
| 删除 | docker rmi imageID |    删除镜像    |



#### 2.容器操作

|   名称   |                      命令                      |                             说明                             |
| :------: | :--------------------------------------------: | :----------------------------------------------------------: |
|   运行   | docker run --name container-name -d image-name | --name：自定义容器名，-d：后台运行，image-name：指定镜像模板 |
|   列表   |                   docker ps                    |              查看运行中的容器，-a：查看所有容器              |
|   停止   |    docker stop container-name/container-id     |                     停止当前你运行的容器                     |
|   启动   |    docker start container-name/container-id    |                           启动容器                           |
|   删除   |             docker rm container-id             |                        s删除指定容器                         |
| 端口映射 |                  -p 6379:6379                  |                -p：主机端口映射到容器内部端口                |

```
docker run -d -p 8888:8080 tomcat
```

-d：后台运行，-p：端口映射





## 3. Dockerfile制作镜像



### 1.FROM指定基础镜像

举例：

```bash
FROM tomcat
```



### 2.RUN执行命令

RUN每条运行的命令都会创建一层新的镜像。 可以使用`&amp;&amp;`进行命令的连接。

#### 1.shell格式

```bash
FROM tomcat
RUN echo “hello docker” &gt; /usr/local/tomcat/webapps/ROOT/index.html
```



### 构建镜像

```bash
docker build -t test .
```



```bash
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM tomcat
 ---&gt; 3639174793ba
Step 2/2 : RUN echo “hello docker” &gt; /usr/local/tomcat/webapps/ROOT/index.html
 ---&gt; Running in c96cba308c1f
Removing intermediate container c96cba308c1f
 ---&gt; caafbd6ff4d7
Successfully built caafbd6ff4d7
Successfully tagged test:latest
```



### COPY指令

```bash
COPY xxx.zip /usr/local/tomcat/webapps/ROOT
RUN unzip xxx.zip
```



### WORKDIR指令

指定一个工作目录

```bash
WORKDIR /usr/local/tomcat/webapps/ROOT
```



#### 构建一个tomcat项目镜像

```bash
FROM tomcat
WORKDIR /usr/local/tomcat/webapps/ROOT

RUN rm -rf *

COPY xxx.zop .

RUN unzip xxx.zip

RUN rm -rf xxx.zip

WORKDIR /usr/local/tomcat
```



构建

```bash
l1nker4@zero:/usr/local/docker/tomcat$ docker build -t xxx .
Sending build context to Docker daemon   38.4kB
Step 1/7 : FROM tomcat
 ---&gt; 3639174793ba
Step 2/7 : WORKDIR /usr/local/tomcat/webapps/ROOT
 ---&gt; Running in ee67259a7a84
Removing intermediate container ee67259a7a84
 ---&gt; 32c64a82f43b
Step 3/7 : RUN rm -rf *
 ---&gt; Running in 9c44abff5efd
Removing intermediate container 9c44abff5efd
 ---&gt; 20a5867132f8
Step 4/7 : COPY xxx.zip .
 ---&gt; c182ed4b79c8
Step 5/7 : RUN unzip xxx.zip
 ---&gt; Running in 445e7367894f
Archive:  xxx.zip
  inflating: rbac.jpg.jpeg           
Removing intermediate container 445e7367894f
 ---&gt; d567880bc000
Step 6/7 : RUN rm -rf xxx.zip
 ---&gt; Running in 2875fbb6a7f8
Removing intermediate container 2875fbb6a7f8
 ---&gt; 08ee1b530f77
Step 7/7 : WORKDIR /usr/local/tomcat
 ---&gt; Running in 96a93d9b9091
Removing intermediate container 96a93d9b9091
 ---&gt; a15ba09df769
Successfully built a15ba09df769
Successfully tagged xxx:latest

```



### ADD指令

和COPY的格式和性质类似，但是有新增的功能。源路径可以是一个`URL`，下载后文件权限设置为`600`（当前用户可读可写，小组无权限，其他组无权限）

如果源路径是一个`tar`压缩文件，可以自动解压。





## 4.Docker守护态运行

同时启动两个tomcat

```bash
docker run -p 8080:8080 --name tomcat -d tomcat
docker run -p 8081:8080 --name tomcat1 -d tomcat
```



## 5.数据卷

- 数据卷是一个或多个容器使用的特殊目录
- 数据卷可以在容器之间共享和重用
- 对数据卷的修改会立刻更新
- 对数据卷的更新，不会影响镜像
- 数据卷默认会一直存在，即使容器删除



问题：容器被销毁，容器中的数据将一并被销毁，容器中的数据不是持久化存在的。



```bash
docker run -p 8080:8080 --name tomcat -d -v /usr/local/docker/tomcat/ROOT:/usr/local/tomcat/webapps/ROOT
```

相当于将宿主机的目录映射到docker容器中目录。



#### 安装MySQL

```bash
docker run -p 3306:3306 --name mysql \
-v /usr/local/docker/conf:/etc/mysql \
-v /usr/local/docker/logs:/var/log/mysql \
-v /usr/local/docker/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:5.7.22
```





## 6.Docker Compose

Docker官方开源项目，Compose定位是定义和运行多个Docker容器的应用。



两个概念

- 服务：一个应用的容器，可以运行多个相同镜像的容器实例。
- 项目：由一组关联的应用容器组成一个完整的业务单元，在`docker-compose.yml`中定义。





### 1.安装Docker-compose

```bash
curl -L https://github.com/docker/compose/releases/download/1.25.0-rc1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

chmod &#43;x /usr/local/bin/docker-compose
```



#### Docker Compose部署Tomcat



`docker-compose.yml`

```yaml
version: &#39;3.1&#39;
services:
  tomcat:
    restart: always
    image: tomcat
    container_name: tomcat
    ports:
      - 8080:8080
    volumes:
      - /usr/local/docker/tomcat/webapps/test:/usr/local/tomcat/webapps/test
    environment:
      TZ: Asia/Shanghai
```



## Reference

[Docker入门教程](http://www.ruanyifeng.com/blog/2018/02/docker-tutorial.html)

[Docker 容器概念](http://jm.taobao.org/2016/05/12/introduction-to-docker)



---

> Author:   
> URL: http://localhost:1313/posts/%E5%B7%A5%E5%85%B7/docker/  

