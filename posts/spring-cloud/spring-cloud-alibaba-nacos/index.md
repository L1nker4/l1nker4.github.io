# Spring Cloud Alibaba Nacos


## 什么是Nacos

&gt; Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。
&gt; Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

Nacos的关键特性包括：
- 服务发现和服务健康监测
- 动态配置服务
- 动态 DNS 服务
- 服务及其元数据管理

## Nacos架构
![Nacos架构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/nacos1.jpeg)

其中Nacos整体分为两大块，分别是`Nacos Server`和`Nacos Console`，`Nacos Server`为核心，其中包括`Naming Service`，`Config Service`，`Naming Service`主要提供服务发现和DNS功能，`Config Service`相当于Netflix时期的Spring Cloud Config提供的分布式配置中心的功能，使得配置信息在线读取。


## Nacos安装

这里笔者使用Docker方式安装，具体安装流程可以参照如下：
```
git clone https://github.com/nacos-group/nacos-docker.git
cd nacos-docker/example
docker-compose -f standalone-mysql-5.7.yaml up -d
```
Nacos 控制台：http://192.168.252.128:8848/nacos

## 服务注册功能案例

1. 引入Nacos Discovery Starter依赖
```xml
 &lt;dependency&gt;
    &lt;groupId&gt;com.alibaba.cloud&lt;/groupId&gt;
    &lt;artifactId&gt;spring-cloud-starter-alibaba-nacos-discovery&lt;/artifactId&gt;
    &lt;version&gt;2.2.0.RELEASE&lt;/version&gt;
 &lt;/dependency&gt;
```

2. 配置Nacos Server地址与其他信息
```
spring.application.name=service-provider
server.port=8081
spring.cloud.nacos.discovery.server-addr=192.168.252.128:8848
```

3. 使用 @EnableDiscoveryClient 注解开启服务注册与发现功能

```java
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudAlibabaNacosExampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudAlibabaNacosExampleApplication.class, args);
    }

}
```

启动应用后，通过Nacos控制台可以看到服务已注册。
![Nacos控制台服务注册情况](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/nacos-register.png)

## 服务发现功能案例

1. 创建Consumer案例，引入Nacos Discovery Starter依赖与Feign依赖
```xml
&lt;dependency&gt;
    &lt;groupId&gt;com.alibaba.cloud&lt;/groupId&gt;
    &lt;artifactId&gt;spring-cloud-starter-alibaba-nacos-discovery&lt;/artifactId&gt;
    &lt;version&gt;2.2.0.RELEASE&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;org.springframework.cloud&lt;/groupId&gt;
    &lt;artifactId&gt;spring-cloud-starter-openfeign&lt;/artifactId&gt;
    &lt;version&gt;2.2.2.RELEASE&lt;/version&gt;
&lt;/dependency&gt;
```

2. 填写相关配置
```
spring.application.name=service-consumer
server.port=8282
spring.cloud.nacos.discovery.server-addr=192.168.252.128:8848
```

3. 添加相关注解
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class SpringCloudAlibabaNaocsConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudAlibabaNaocsConsumerApplication.class, args);
    }

}
```


4. 编写通过Feign调用服务代码

```java
@FeignClient(name = &#34;service-provider&#34;)
public interface EchoService {
    
    @GetMapping(value = &#34;/echo/{str}&#34;)
    String echo(@PathVariable(&#34;str&#34;) String str);
    
}
```

```java
@RestController
public class TestController {

    @Autowired
    private EchoService echoService;

    @GetMapping(value = &#34;/feign/{str}&#34;)
    public String feign(@PathVariable String str) {
        return echoService.echo(str);
    }
}
```
通过访问：http://localhost:8282/feign/xxx

可以看出，成功通过Feign调用service-provider服务

![调用结果](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/res.png)


## 负载均衡案例

首先修改controller代码。
```java
@RestController
public class EchoController {

    @Value(&#34;${server.port}&#34;)
    private String port;

    @GetMapping(&#34;echo/{str}&#34;)
    public String echo(@PathVariable String str) {
        return &#34;Nacos Discovery &#34; &#43; str &#43; &#34; from port: &#34; &#43; port;
    }
}
```

开启允许多个实例运行的配置：
![运行多个实例配置](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/run.png)

将配置文件中运行端口改为8083，Nacos控制台如图所示：
![Nacos控制台情况](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/demo.png)

可以看出，provider运行了两个实例。下面通过消费者服务进行服务调用。
![运行结果1](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/demo1.png)

![运行结果2](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/demo2.png)

## 配置中心案例

1. 创建项目，导入Nacos Config Server依赖。添加`@EnableDiscoveryClient`注解
```xml
 &lt;dependency&gt;
    &lt;groupId&gt;com.alibaba.cloud&lt;/groupId&gt;
    &lt;artifactId&gt;spring-cloud-starter-alibaba-nacos-config&lt;/artifactId&gt;
 &lt;/dependency&gt;
 &lt;dependency&gt;
    &lt;groupId&gt;com.alibaba.cloud&lt;/groupId&gt;
    &lt;artifactId&gt;spring-cloud-starter-alibaba-nacos-discovery&lt;/artifactId&gt;
 &lt;/dependency&gt;
```

2. 创建配置文件`bootstrap.properties`，填写如下配置：
```
spring.application.name=nacos-config-example
server.port=18084
spring.cloud.nacos.config.server-addr=192.168.252.128:8848
```

3. 在Nacos控制台新添加如下配置：
![Nacos添加配置](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/config.png)

4. 编写Controller代码
```java
@RestController
public class TestController {

    @Value(&#34;${user.id}&#34;)
    private String id;

    @Value(&#34;${user.name}&#34;)
    private String name;

    @GetMapping(&#34;test&#34;)
    public String test() {
        return id &#43; &#34;, Hello &#34;&#43; name;
    }
}
```

5. 访问：http://localhost:18084/test
![访问结果](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/spring-cloud/nacos/res1.png)
可以看到，通过Nacos配置中心，成功读取到配置文件的内容。


下面列出Nacos的其它配置项：


| 配置项         |                      key                       | 默认值                  | 说明                                                         |
| :------------- | :--------------------------------------------: | ----------------------- | ------------------------------------------------------------ |
| 服务端地址     |    spring.cloud.nacos.discovery.server-addr    |                         |                                                              |
| 服务名         |      spring.cloud.nacos.discovery.service      | spring.application.name |                                                              |
| 权重           |      spring.cloud.nacos.discovery.weight       | 1                       | 取值范围 1 到 100，数值越大，权重越大                        |
| 网卡名         | spring.cloud.nacos.discovery.network-interface |                         | 当IP未配置时，注册的IP为此网卡所对应的IP地址，如果此项也未配置，则默认取第一块网卡的地址 |
| 注册的IP地址   |        spring.cloud.nacos.discovery.ip         |                         | 优先级最高                                                   |
| 注册的端口     |       spring.cloud.nacos.discovery.port        | -1                      | 默认情况下不用配置，会自动探测                               |
| 命名空间       |     spring.cloud.nacos.discovery.namespace     |                         | 常用场景之一是不同环境的注册的区分隔离，例如开发测试环境和生产环境的资源（如配置、服务）隔离等。 |
| AccessKey      |    spring.cloud.nacos.discovery.access-key     |                         |                                                              |
| SecretKey      |    spring.cloud.nacos.discovery.secret-key     |                         |                                                              |
| Metadata       |     spring.cloud.nacos.discovery.metadata      |                         | 使用Map格式配置                                              |
| 日志文件名     |     spring.cloud.nacos.discovery.log-name      |                         |                                                              |
| 接入点         |     spring.cloud.nacos.discovery.endpoint      | UTF-8                   | 地域的某个服务的入口域名，通过此域名可以动态地拿到服务端地址 |
| 是否集成Ribbon |              ribbon.nacos.enabled              | true                    |                                                              |

---

> Author:   
> URL: http://localhost:1313/posts/spring-cloud/spring-cloud-alibaba-nacos/  

