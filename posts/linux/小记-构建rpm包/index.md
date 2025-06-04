# 小记-构建RPM包




## 准备工作



OS环境与rpm版本号如下：

```
[root@localhost ~]# uname -a
Linux localhost.localdomain 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux

[root@localhost ~]# rpm --version
RPM 版本 4.11.3
```



安装rpm-build

```
yum install -y rpm-build	# 构建的rpm的核心组件
yum install -y rpmdevtools  # 提供了构建rpm包的一些工具
```



查看工作目录：

```
[root@localhost rpmbuild]# rpmbuild --showrc | grep topdir
-14: _builddir	%{_topdir}/BUILD
-14: _buildrootdir	%{_topdir}/BUILDROOT
-14: _rpmdir	%{_topdir}/RPMS
-14: _sourcedir	%{_topdir}/SOURCES
-14: _specdir	%{_topdir}/SPECS
-14: _srcrpmdir	%{_topdir}/SRPMS
-14: _topdir	%{getenv:HOME}/rpmbuild
```



工作目录位于`$HOME/rpmbuild`，在该目录下创建以下文件夹、或直接使用`rpmdev-setuptree`生成目录：

```
[root@localhost rpmbuild]# mkdir -pv /root/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
```



目录的各个用途如下：

- `BUILD`：目录用来存放打包过程中的源文件，就是来源于 `SOURCE`
- `BUILDROOT`：编译后存放的目录
- `SOURCE` ：用来存放打包是要用到的源文件和 patch补丁，主要是一些 `tar` 包
- `SPECS`：用来存放 spec 文件，spec是构建RPM包的配置文件
- `RPMS`：存放打包生成的rpm二进制包
- `SRPMS`：存放打包生成的源码RPM包





## SPEC

spec文件是构建RPM包的核心文件，内部包含了软件包的名称、版本、编译指令、安装指令、卸载指令、依赖文件等。

可以使用`rpmdev-newspec -o name.spec`来生成SPEC模板文件。



### 软件包信息

该部分包含了如下信息，可通过`%{value}`在后续进行引用：

- Name：软件包名称
- Version：软件版本
- Summary：概述
- Group：软件分组，通过`cat /usr/share/doc/rpm-VERSION/GROUPS`查看详细分组
- Source0、Source1等：源码包位置，位于`%_topdir/SOURCE`文件夹下。



### 依赖

- BuildRequires：定义构建时需要的软件包，逗号分隔，例如`gcc &gt;=4.2.2`
- Requires：安装时的依赖包。格式如下：
  - `Requires:       xx_package &gt;= %{version}`



### 编译

- BuildRoot：定义了`make install`的测试安装目录，默认指定为`%_topdir/BUILDROOT`
- BuildArch：编译时处理器架构，默认从`cat /usr/lib/rpm/macros`读取



## 阶段



### description

该阶段包含对rpm包的描述，可以通过`rpm -qpi`读取。

```
[root@localhost rpmbuild]# rpm -qpi RPMS/x86_64/bda-monitor-3.0.0-1.x86_64.rpm 
Name        : bda-monitor
Version     : 3.0.0
Release     : 1
Architecture: x86_64
Install Date: (not installed)
Group       : Applications/Server
Size        : 644017273
License     : Commercial License
Signature   : (none)
Source RPM  : bda-monitor-3.0.0-1.src.rpm
Build Date  : 2022年06月21日 星期二 03时34分39秒
Build Host  : localhost
Relocations : /opt/bda-monitor 
Summary     : 业务处置平台-监控模块
Description :
```





### prep

预处理阶段，该部分填写准备阶段的shell脚本，可以清空指定文件夹，对源码包进行解压等操作。

例如：

```
%prep
rm -rf $RPM_BUILD_DIR/openssl-1.1.1n
tar fx $RPM_SOURCE_DIR/openssl-1.1.1n.tar.gz
```



### build

该阶段主要对`%_builddir`目录下的源码包执行`./configure`和`make`。

例如：

```
cd $RPM_BUILD_DIR/nginx-1.20.1
./configure \
--prefix=%{prefix}/nginx \
--with-http_ssl_module \
--with-openssl=$RPM_BUILD_DIR/openssl-1.1.1n \
--with-http_gzip_static_module \
--with-http_auth_request_module \
--with-http_stub_status_module

make %{?_smp_mflags}
```





### install

该部分就是执行`make install`的操作，将编译好的二进制文件安装到虚拟目录中（或者直接将二进制文件install），这个阶段会在`%buildrootdir` 目录里建好目录，然后将需要打包的文件从 `%builddir` 里拷贝到 `%_buildrootdir` 里。

```
# prometheus
%ifarch x86_64
cp -pR prometheus/x86_64/* %{buildroot}%{prefix}/prometheus/
%else
cp -pR prometheus/aarch64/* %{buildroot}%{prefix}/prometheus/
%endif

# grafana
%ifarch x86_64
cp -pR grafana/x86_64/* %{buildroot}%{prefix}/grafana/
%else
cp -pR grafana/aarch64/* %{buildroot}%{prefix}/grafana/
%endif
```





### files

文件段，用来说明`%{buildroot}`目录下的哪些文件目录最终打包到rpm包中。

```
%defattr(文件权限,用户名,组名,目录权限) 	# 设置文件权限
%config(noreplace) /etc/my.cnf         # 表明是配置文件，noplace表示替换文件
%attr(644, root, root) %{_mandir}/man8/mysqld.8* # 分别是权限，属主，属组

```





### 其他阶段

- pre：安装前执行的阶段
- post：安装后执行的阶段
- preun：卸载前执行的阶段
- postun：卸载后执行的阶段
- changelog：日志段
- clean：打包后的清理段，主要对`%{buildroot}`进行清空





## spec模板

```
# 软件包的基本信息
Name:           bda-monitor
Version:        3.0.0
Release:        1
Summary:        业务处置平台-监控模块
License:        Commercial License
Group: 			Applications/Server

# 将要安装的目录位置
prefix: /opt/bda-monitor

# 描述信息
%description

# 准备部分
%prep

# 安装前执行
%pre
echo -e &#39;开始安装业务处置平台-监控模块&#39;


# 编译 
%build

# 安装
%install

cd $RPM_BUILD_DIR
mkdir -p %{buildroot}%{prefix}/mysqld_exporter
mkdir -p %{buildroot}%{prefix}/redis_exporter
mkdir -p %{buildroot}%{prefix}/blackbox_exporter
mkdir -p %{buildroot}%{prefix}/elasticsearch_exporter
mkdir -p %{buildroot}%{prefix}/kafka_exporter
mkdir -p %{buildroot}%{prefix}/node_exporter
mkdir -p %{buildroot}%{prefix}/postgres_exporter
mkdir -p %{buildroot}%{prefix}/rabbitmq_exporter
mkdir -p %{buildroot}%{prefix}/zookeeper_exporter
mkdir -p %{buildroot}%{prefix}/grafana
mkdir -p %{buildroot}%{prefix}/prometheus

# mysqld_exporter

# ifarch 判断指令集，安装相应指令集的source
%ifarch x86_64
cp -pR mysqld_exporter/x86_64/* %{buildroot}%{prefix}/mysqld_exporter/
%else
cp -pR mysqld_exporter/aarch64/* %{buildroot}%{prefix}/mysqld_exporter/
%endif


# redis_exporter
%ifarch x86_64
cp -pR redis_exporter/x86_64/* %{buildroot}%{prefix}/redis_exporter/
%else
cp -pR redis_exporter/aarch64/* %{buildroot}%{prefix}/redis_exporter/
%endif

# blackbox_exporter
%ifarch x86_64
cp -pR blackbox_exporter/x86_64/* %{buildroot}%{prefix}/blackbox_exporter/
%else
cp -pR blackbox_exporter/aarch64/* %{buildroot}%{prefix}/blackbox_exporter/
%endif

# elasticsearch_exporter
%ifarch x86_64
cp -pR elasticsearch_exporter/x86_64/* %{buildroot}%{prefix}/elasticsearch_exporter/
%else
cp -pR elasticsearch_exporter/aarch64/* %{buildroot}%{prefix}/elasticsearch_exporter/
%endif

# kafka_exporter
%ifarch x86_64
cp -pR kafka_exporter/x86_64/* %{buildroot}%{prefix}/kafka_exporter/
%else
cp -pR kafka_exporter/aarch64/* %{buildroot}%{prefix}/kafka_exporter/
%endif

# node_exporter
%ifarch x86_64
cp -pR node_exporter/x86_64/* %{buildroot}%{prefix}/node_exporter/
%else
cp -pR node_exporter/aarch64/* %{buildroot}%{prefix}/node_exporter/
%endif

# postgres_exporter
%ifarch x86_64
cp -pR postgres_exporter/x86_64/* %{buildroot}%{prefix}/postgres_exporter/
%else
cp -pR postgres_exporter/aarch64/* %{buildroot}%{prefix}/postgres_exporter/
%endif

# rabbitmq_exporter
%ifarch x86_64
cp -pR rabbitmq_exporter/x86_64/* %{buildroot}%{prefix}/rabbitmq_exporter/
%else
cp -pR rabbitmq_exporter/aarch64/* %{buildroot}%{prefix}/rabbitmq_exporter/
%endif

# zookeeper_exporter
%ifarch x86_64
cp -pR zookeeper_exporter/x86_64/* %{buildroot}%{prefix}/zookeeper_exporter/
%else
cp -pR zookeeper_exporter/aarch64/* %{buildroot}%{prefix}/zookeeper_exporter/
%endif

# prometheus
%ifarch x86_64
cp -pR prometheus/x86_64/* %{buildroot}%{prefix}/prometheus/
%else
cp -pR prometheus/aarch64/* %{buildroot}%{prefix}/prometheus/
%endif

# grafana
%ifarch x86_64
cp -pR grafana/x86_64/* %{buildroot}%{prefix}/grafana/
%else
cp -pR grafana/aarch64/* %{buildroot}%{prefix}/grafana/
%endif

# 安装后执行、运行系统命令、重启服务等初始化行为
%post

# 卸载完成后执行的脚本
%postun
rm -rf %{prefix}

# 清理阶段，构建完成rpm包后，可以删除build过程的中间文件
%clean
rm -rf %{buildroot}


# 此部分定义了要安装的文件以及在目录树中的位置
%files
%{prefix}


%changelog
# 1. 创建监控模块所需各类exporter &#43; Prometheus &#43; Grafana

```





## 打包



当SPEC文件编写完成后，并且`SOURCES`目录下的源码包或`BUILD`目录下的二进制文件准备好，执行如下的打包指令：

```
rpmbuild --target=x86_64 -bb SPECS/bda-monitor.spec
```



参数说明如下：

```
-bp 执行 spec 文件的&#34;%prep&#34;阶段。这通常等价于解包源代码并应用补丁。
-bc 执行 spec 文件的&#34;%build&#34;阶段(在执行了 %prep 之后)。这通常等价于执行了&#34;make&#34;。
-bi 执行 spec 文件的&#34;%install&#34;阶段(在执行了 %prep, %build 之后)。这通常等价于执行了&#34;make install&#34;。
-bb 构建二进制包(在执行 %prep, %build, %install 之后)
-bs 只构建源代码包
-ba 构建二进制包和源代码包(在执行 %prep, %build, %install 之后)
```



---

> Author:   
> URL: http://localhost:1313/posts/linux/%E5%B0%8F%E8%AE%B0-%E6%9E%84%E5%BB%BArpm%E5%8C%85/  

