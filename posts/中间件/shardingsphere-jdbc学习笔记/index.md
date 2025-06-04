# ShardingSphere-JDBC学习笔记






## 基础概念

ShardingSphere-JDBC是Apache ShardingSphere项目中的一个子项目，Apache ShardingSphere是一款分布式的数据库生态系统，可以通过分片、弹性伸缩、加密等能力对原有数据库进行增强。

ShardingSphere-JDBC定位是**轻量级Java框架**，在JDBC层提供额外服务。它能尽量透明化水平分库分表所带来的影响，让业务方逻辑上感知到一个数据库节点和逻辑表，当收到SQL查询，主要做了以下工作：

- SQL解析：词法解析和语法解析，提炼出解析上下文
- SQL路由：根据解析上下文匹配用户配置的库表的分片策略，并生成路由后的SQL（一条或多条）。
- SQL改写：将SQL改写为物理数据库能正常执行的语句（逻辑SQL -&gt; 物理SQL）。
- SQL执行：通过多线程异步执行改写后的SQL语句。
- 结果归并：将多个执行结果归并为统一的JDBC接口输出。



几个概念：

- 逻辑表：ORM框架的业务层面，表现为一张表，例如：t_order
- 物理表：数据库层面实际存在的表，例如：t_order_0、t_order_1
- 绑定表：分片规则一致的一组分片表，进行关联查询时，建议使用分片键进行关联，否则影响查询效率。
  - `SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);`
- 广播表：数据源中都存在的表，且结构和数据都完全一致。适用于数据量不大且需要与其他大数据量表进行关联查询。

- 分片键：根据某个字段的计算结果（取模等）进行水平分片
- 路由：通过SQL语句中的信息，找到对应分片的过程



### 分片算法

- 标准分片算法：单一键作为分片键。

  - 取模算法：根据一些字段，或多个字段hash求值再取模。

  - 范围限定算法，按照年份、实践等策略路由到目标数据库。


- 复合分片算法：多键作为分片键，自行设计
- Hint分片算法：用于处理使用Hint行分片的场景（非数据库字段的分片方式）



### 分片策略



包含分片键和分片算法，ShardingSphere-JDBC提供了以下几种分片策略：

- 标准分片策略（StandardSharingStrategy）：使用精确分片算法或范围分片算法，支持单分片键。
- 复合分片策略（ComplexShardingStrategy）：使用复合分片算法，支持多分片键。
- Hint分片策略（HintShardingStrategy）：使用Hint分片算法
- Inline分片策略（InlineShardingStrategy）：使用groovy表达式作为分片算法
- 不分片策略（NoneShardingStrategy）：不使用分片算法





## Demo



pom.xml

```xml
&lt;dependency&gt;
            &lt;groupId&gt;org.mybatis.spring.boot&lt;/groupId&gt;
            &lt;artifactId&gt;mybatis-spring-boot-starter&lt;/artifactId&gt;
            &lt;version&gt;2.1.4&lt;/version&gt;
        &lt;/dependency&gt;

        &lt;dependency&gt;
            &lt;groupId&gt;com.mysql&lt;/groupId&gt;
            &lt;artifactId&gt;mysql-connector-j&lt;/artifactId&gt;
            &lt;scope&gt;runtime&lt;/scope&gt;
            &lt;version&gt;8.0.32&lt;/version&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;org.projectlombok&lt;/groupId&gt;
            &lt;artifactId&gt;lombok&lt;/artifactId&gt;
            &lt;optional&gt;true&lt;/optional&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;com.baomidou&lt;/groupId&gt;
            &lt;artifactId&gt;mybatis-plus-boot-starter&lt;/artifactId&gt;
            &lt;version&gt;3.2.0&lt;/version&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;com.alibaba&lt;/groupId&gt;
            &lt;artifactId&gt;druid&lt;/artifactId&gt;
            &lt;version&gt;1.2.15&lt;/version&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
            &lt;artifactId&gt;spring-boot-starter-test&lt;/artifactId&gt;
            &lt;scope&gt;test&lt;/scope&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;org.apache.shardingsphere&lt;/groupId&gt;
            &lt;artifactId&gt;sharding-jdbc-spring-boot-starter&lt;/artifactId&gt;
            &lt;version&gt;4.0.0-RC1&lt;/version&gt;
        &lt;/dependency&gt;

        &lt;dependency&gt;
            &lt;groupId&gt;org.apache.shardingsphere&lt;/groupId&gt;
            &lt;artifactId&gt;sharding-core-common&lt;/artifactId&gt;
            &lt;version&gt;4.0.0-RC1&lt;/version&gt;
        &lt;/dependency&gt;
```



ShardingSphere提供了多种配置方式：

- Java代码配置
- yaml、properties配置
- Spring Boot配置



该案例使用配置文件方式：application.properties

```
spring.shardingsphere.datasource.names=test-0,test-1

spring.main.allow-bean-definition-overriding=true
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.shardingsphere.datasource.test-0.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.test-0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.test-0.url=jdbc:mysql://172.27.184.50:3306/test-0?useUnicode=true&amp;characterEncoding=utf8&amp;tinyInt1isBit=false&amp;useSSL=false
spring.shardingsphere.datasource.test-0.username=root
spring.shardingsphere.datasource.test-0.password=123456

spring.shardingsphere.datasource.test-1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.test-1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.test-1.url=jdbc:mysql://172.27.184.50:3306/test-1?useUnicode=true&amp;characterEncoding=utf8&amp;tinyInt1isBit=false&amp;useSSL=false
spring.shardingsphere.datasource.test-1.username=root
spring.shardingsphere.datasource.test-1.password=123456

spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=test-$-&gt;{order_id % 2}

spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=test-$-&gt;{0..1}.t_order_$-&gt;{0..2}
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$-&gt;{order_id % 3}


spring.shardingsphere.sharding.broadcast-tables=t_config
spring.shardingsphere.props.sql.show=true
```



分别创建`test-0`和`test-1`数据库，并创建以下表：

```sql
CREATE TABLE `t_order_0` (
  `order_id` bigint(200) NOT NULL,
  `order_no` varchar(100) DEFAULT NULL,
  `create_name` varchar(50) DEFAULT NULL,
  `price` decimal(10,2) DEFAULT NULL,
  PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;


CREATE TABLE `t_order_1` (
  `order_id` bigint(200) NOT NULL,
  `order_no` varchar(100) DEFAULT NULL,
  `create_name` varchar(50) DEFAULT NULL,
  `price` decimal(10,2) DEFAULT NULL,
  PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;


CREATE TABLE `t_order_2` (
  `order_id` bigint(200) NOT NULL,
  `order_no` varchar(100) DEFAULT NULL,
  `create_name` varchar(50) DEFAULT NULL,
  `price` decimal(10,2) DEFAULT NULL,
  PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC;



CREATE TABLE `t_config` (
  `id` bigint(30) NOT NULL,
  `remark` varchar(50) CHARACTER SET utf8 DEFAULT NULL,
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `last_modify_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```



![数据库结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20230212192526669.png)



```java
@Data
@TableName(&#34;t_order&#34;)
@ToString
public class TOrder {

    private Long orderId;

    private String orderNo;

    private String createName;

    private BigDecimal price;
}

@Service
public class OrderService {

    @Resource
    private OrderMapper orderMapper;

    private static final AtomicLong ID = new AtomicLong(1);

    public void insertOrder() {
        for (int i = 0; i &lt; 1000; i&#43;&#43;) {
            TOrder order = new TOrder();
            order.setOrderId(ID.getAndIncrement());
            order.setOrderNo(&#34;A000&#34; &#43; i);
            order.setCreateName(&#34;订单 &#34; &#43; i);
            order.setPrice(new BigDecimal(&#34;&#34; &#43; i));
            orderMapper.insert(order);
        }
    }

    public List&lt;TOrder&gt; selectList(){
        QueryWrapper&lt;TOrder&gt; queryWrapper = new QueryWrapper&lt;&gt;();
        queryWrapper.like(&#34;create_name&#34;, &#34;订单&#34;);
        return orderMapper.selectList(queryWrapper);
    }

}

@Mapper
public interface OrderMapper extends BaseMapper&lt;TOrder&gt; {
    
}

@Component
@Slf4j
public class Runner implements ApplicationRunner {

    @Resource
    private OrderService orderService;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        orderService.insertOrder();
//        List&lt;TOrder&gt; orderList = orderService.selectList();
//        log.info(&#34;orderList : {}&#34;, orderList);
    }
}
```



查询数据库，数据按照指定规则存储：

![数据库结果](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20230212210352405.png)





---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/shardingsphere-jdbc%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/  

