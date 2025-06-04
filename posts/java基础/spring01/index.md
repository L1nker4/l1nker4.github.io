# Spring学习笔记（一）




# 基础



## Spring介绍

1. Spring是一个开源框架。
2. Spring为简化企业级应用开发而生，使用Spring可以使简单的JavaBean实现以前只有EJB才能实现的功能。
3. Spring是一个IOC(DI)和AOP容器框架。
4. 轻量级：Spring是非侵入式的-基于Spring开发的应用中的对象可以不依赖于Spring的API
5. 依赖注入(DI)：Dependency Injection，IOC
6. 面向切面编程（AOP-aspect oriented programming）
7. 容器：Spring是一个容器，因为它包含并且管理应用对象的生命周期
8. 框架：Spring实现了使用简单的组件配置组合成一个复杂的应用，在Spring中可以使用XML和Java注解组合这些对象。
9. 一站式：在IOC和AOP的基础上可以整合各种企业应用的开源框架和优秀的第三方类库



## 核心特性

- IoC容器
- Spring事件
- 资源管理
- 国际化
- 校验
- 数据绑定
- 类型转换
- Spring表达式
- 面向切面编程





### IOC和DI

IOC（Inversion of Control）：其思想是反转资源获取的方向。

传统的资源查找方式要求组件向容器发起请求查找资源，作为回应，容器适时的返回资源，而应用了IOC之后，则是容器主动地将资源推送给它所管理的组件，组件所要做的仅是选择一种合适的方式来接受资源，这种行为 也被称为查找的被动形式。



IoC容器的职责：

- 依赖处理
  - 依赖查找
  - 依赖注入
- 生命周期管理
  - 容器
  - 托管的资源
- 配置
  - 容器
  - 外部化配置
  - 托管资源





DI：IOC的另一种表述方式，即组件以一些预先定义好的方式（例如：setter方法）接受来自如容器的资源注入，相对于IOC而言，这种表述更直接。



### 配置Bean

配置形式：基于XML文件的方式，基于注解的方式

Bean的配置方式：通过全类名（反射），通过工厂方法（静态工厂方法&amp;实例工厂方法），FactoryBean



```xml
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;
       xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd&#34;&gt;

    &lt;!--配置Bean--&gt;
    &lt;bean id=&#34;helloWorld&#34; class=&#34;HelloWorld&#34;&gt;
        &lt;property name=&#34;name&#34; value=&#34;Spring&#34;/&gt;
    &lt;/bean&gt;
&lt;/beans&gt;
```

class：bean的全类名，通过反射的方式在IOC容器中创建bean，所以要求bean要有无参构造器

id：标识容易中的bean，id唯一





Spring提供了两种类型的IOC容器实现：

- BeanFactory：IOC容器的基本实现
- ApplicationContext 提供了更多的高级特大型，是BeanFactory的子接口



BeanFactory是Spring框架的基础设施，面向Spring本身

ApplicationContext 面向使用Spring框架的开发者，几乎所有的应用场合都直接使用ApplicationContext



```java
ApplicationContext ctx = new ClassPathXmlApplicationContext(&#34;applicationContext.xml&#34;);
HelloWorld helloWorld = (HelloWorld) ctx.getBean(&#34;helloWorld&#34;);
```

利用id定位到IOC容器中的Bean



ApplicationContext 代表IOC容器



ApplicationContext的两个实现类

ClassPathXmlApplicationContext 是从类路径下加载配置文件

FileSystemXmlApplicationContext从文件系统中加载配置文件



### 依赖注入的方式



#### 属性注入(最常用)

即通过setter方法注入Bean的属性值或依赖的对象

属性注入使用`&lt;property&gt;`元素，使用name属性指定bean的属性名称，value属性，或者value子节点指定属性值

属性注入是实际应用中最常用的注入方式

```xml
&lt;property name=&#34;name&#34; value=&#34;Spring&#34;/&gt;
```



#### 构造方法注入

通过构造方法注入Bean的属性值或者依赖的对象，他保证了Bean实例在实例化后就可以使用。

构造器注入在`&lt;constructor-arg&gt;元素里声明属性`

`&lt;constructor-arg&gt;`中没有name属性

```xml
&lt;!--可以指定参数的位置和参数的类型，以区分重载的构造器--&gt;
    &lt;bean id=&#34;car2&#34; class=&#34;Car&#34;&gt;
        &lt;constructor-arg value=&#34;BMW&#34; type=&#34;java.lang.String&#34;/&gt;
        &lt;constructor-arg value=&#34;Shanghai&#34; type=&#34;java.lang.String&#34;/&gt;
        &lt;constructor-arg value=&#34;240&#34; type=&#34;int&#34;/&gt;
    &lt;/bean&gt;
```

```xml
&lt;constructor-arg type=&#34;int&#34;&gt;
    &lt;value&gt;240&lt;/value&gt;
&lt;/constructor-arg&gt;
```

若注入的值含有特殊字符，可以使用`&lt;![CDATA[]]&gt;将字面值包起来`

```xml
&lt;constructor-arg type=&#34;int&#34;&gt;
    &lt;value&gt;&lt;![CDATA[&lt;shanghai^]]&gt;&lt;/value&gt;
&lt;/constructor-arg&gt;
```



#### 工厂方式注入

（很少使用，不推荐）



##### 在Bean中引用其他Bean

（ref元素）

```xml
&lt;bean id=&#34;person&#34; class=&#34;Preson&#34;&gt;
        &lt;property name=&#34;name&#34; value=&#34;Tom&#34;/&gt;
        &lt;property name=&#34;age&#34; value=&#34;24&#34;/&gt;
        &lt;property name=&#34;car&#34; ref=&#34;car2&#34;/&gt;
    &lt;/bean&gt;
```



也可以在属性或构造器里包含Bean的声明

```xml
&lt;!--内部bean，不能被外部引用，只能在内部使用--&gt;
&lt;property name=&#34;car&#34;&gt;
            &lt;bean class=&#34;Car&#34;&gt;
                &lt;constructor-arg value=&#34;ford&#34;/&gt;
                &lt;constructor-arg value=&#34;Chanan&#34;/&gt;
                &lt;constructor-arg value=&#34;200000&#34; type=&#34;double&#34;/&gt;
            &lt;/bean&gt;
        &lt;/property&gt;
```





#### null 值和级联属性

```xml
&lt;constructor-arg&gt;&lt;null/&gt;&lt;/constructor-arg&gt;
```



级联属性
```xml
&lt;property name=&#34;car.price&#34; value=&#34;20000&#34;/&gt;
```

为级联属性赋值：属性需要先初始化后才能为级联属性赋值，否则会有异常。





### 集合属性

```xml
&lt;!--测试配置集合属性--&gt;
    &lt;bean id=&#34;person3&#34; class=&#34;com.test.spring.collection.Person&#34;&gt;
        &lt;property name=&#34;name&#34; value=&#34;Mike&#34;/&gt;
        &lt;property name=&#34;age&#34; value=&#34;27&#34;/&gt;
        &lt;property name=&#34;cars&#34;&gt;
            &lt;list&gt;
                &lt;ref bean=&#34;car&#34;/&gt;
                &lt;ref bean=&#34;car2&#34;/&gt;
            &lt;/list&gt;
        &lt;/property&gt;
    &lt;/bean&gt;
```

数组也用`&lt;list&gt;`

Set使用`&lt;set&gt;`



Map类型

```xml
&lt;!--配置map属性值--&gt;
    &lt;bean id=&#34;newPerson&#34; class=&#34;com.test.spring.collection.NewPerson&#34;&gt;
        &lt;property name=&#34;name&#34; value=&#34;rose&#34;/&gt;
        &lt;property name=&#34;age&#34; value=&#34;28&#34;/&gt;
        &lt;property name=&#34;cars&#34;&gt;
            &lt;map&gt;
                &lt;entry key=&#34;AA&#34; value-ref=&#34;car&#34;/&gt;
                &lt;entry key=&#34;BB&#34; value-ref=&#34;car2&#34;/&gt;
            &lt;/map&gt;
        &lt;/property&gt;
```





Properties类型

```xml
&lt;!--配置Properties属性值--&gt;
    &lt;bean id=&#34;dataSource&#34; class=&#34;com.test.spring.collection.DataSource&#34;&gt;
        &lt;property name=&#34;properties&#34;&gt;
            &lt;props&gt;
                &lt;prop key=&#34;user&#34;&gt;root&lt;/prop&gt;
                &lt;prop key=&#34;password&#34;&gt;123456&lt;/prop&gt;
                &lt;prop key=&#34;jdbcUrl&#34;&gt;jdbc:mysql://xxx&lt;/prop&gt;
                &lt;prop key=&#34;DriverClass&#34;&gt;com.mysql.jdbc.Driver&lt;/prop&gt;

            &lt;/props&gt;
        &lt;/property&gt;
    &lt;/bean&gt;
```



#### p命名空间

通过p命名空间为bean的属性赋值，需要先导入p命名空间

```xml
&lt;bean id=&#34;person5&#34; class=&#34;com.test.spring.collection.Person&#34; p:age=&#34;30&#34; p:name=&#34;Queen&#34; p:cars-ref=&#34;cars&#34;/&gt;
```





### Spring自动装配

在`&lt;bean&gt;`的autowire属性里面指定自动装配的模式。



##### 原始方式

```xml
&lt;bean id=&#34;address&#34; class=&#34;com.test.spring.autowire.Address&#34; p:city=&#34;Beijing&#34; p:street=&#34;tiananmen&#34;/&gt;

&lt;bean id=&#34;car&#34; class=&#34;com.test.spring.autowire.Car&#34; p:brand=&#34;Audi&#34; p:price=&#34;300000&#34;/&gt;

&lt;bean id=&#34;person&#34; class=&#34;com.test.spring.autowire.Person&#34; p:name=&#34;Tom&#34; p:address-ref=&#34;address&#34; p:car-ref=&#34;car&#34;/&gt;
```





##### 自动装配

```xml
&lt;bean id=&#34;person&#34; class=&#34;com.test.spring.autowire.Person&#34; p:name=&#34;Tom&#34; autowire=&#34;byName&#34;/&gt;
```

**使用autowire属性指定自动装配的方式**

byName：根据bean的名字和当前bean的setter风格的属性名进行自动装配

ByType：根据bean的类型和当前bean的属性的类型进行自动装配，若IOC容器中有一个以上的类型匹配的Bean，则抛异常。



##### 自动装配的缺点

不够灵活，autowire要么根据类型自动匹配，要么根据名称自动匹配，不能兼有。

实际的项目中很少使用自动装配





### Bean之间的关系



#### 继承

```xml
&lt;bean id=&#34;address&#34; class=&#34;com.test.spring.autowire.Address&#34; p:city=&#34;BeiJing&#34; p:street=&#34;WuDaoKou&#34;/&gt;
&lt;bean id=&#34;address2&#34; class=&#34;com.test.spring.autowire.Address&#34; p:street=&#34;DaZongShi&#34; parent=&#34;address&#34;/&gt;
```



- Spring允许继承Bean的配置
- 子Bean从父Bean中继承配置，包括Bean的属性配置
- 子Bean也可以覆盖从父Bean继承过来的配置
- 父Bean可以作为配置模板，也可以作为Bean实例，若想把父Bean作为模板，可以**设置`&lt;bean&gt;`的abstract属性为True**，这样Spring将不会实例化这个Bean，这个抽象Bean就是用来继承的。
- 并不是Bean元素里面所有属性都会被继承，比如：autowire，abstract等
- 也可以忽略父Bean的class属性，让子Bean指定自己的类，而共享相同的属性配置，但此时必须设为true





#### 依赖


需求：在配置Person时，必须有一个关联的car，换句话说person这个Bean依赖于car这个Bean

```xml
&lt;bean id=&#34;car&#34; class=&#34;com.test.spring.autowire.Car&#34; p:brand=&#34;Audi&#34; p:price=&#34;300000&#34;/&gt;
&lt;bean id=&#34;person&#34; class=&#34;com.test.spring.autowire.Person&#34; p:name=&#34;tom&#34; p:address-ref=&#34;address&#34; depends-on=&#34;car&#34;/&gt;

```

- Spring允许用户通过depends-on属性设定Bean前置依赖的Bean，前置依赖的Bean会在本Bean实例化之前创建好。
- 如果前置依赖于多个Bean，则可以通过逗号，空格的方式配置Bean的名称





### Bean的作用域

在 Spring 中, 可以在 `&lt;bean&gt; `元素的 scope 属性里设置 Bean 的作用域. 

singleton：默认值，容器初始化时创建bean实例，在整个容器的生命周期内只创建这一个bean，单例的。

prototype：原型的，容器初始化时不创建bean实例 ，而在每次请求时都创建一个新的Bean实例，并返回





### 使用外部属性文件

在配置文件里面配置Bean的时候，有时候需要在Bean的配置里面混入系统部署的细节信息（文件路径，数据源配置信息），而这些部署细节实际上需要和Bean配置相分离。

```xml
		&lt;!--导入属性文件--&gt;
    	&lt;context:properties location=&#34;db.properties&#34;/&gt;
    	&lt;!--使用外部属性文件的属性--&gt;
        &lt;bean id=&#34;dataSource&#34; class=&#34;com.mchange.v2.c3p0.ComboPooledDataSource&#34;&gt;
            &lt;property name=&#34;user&#34; value=&#34;${user}&#34;/&gt;
            &lt;property name=&#34;password&#34; value=&#34;${password}&#34;/&gt;
            &lt;property name=&#34;driverClass&#34; value=&#34;${driverclass}&#34;/&gt;
            &lt;property name=&#34;jdbcUrl&#34; value=&#34;${jdbcUrl}&#34;/&gt;
		&lt;/bean&gt;
```







### SpEL

- Spring表达式语言：是一个支持运行时查询和操作对象圈的强大的表达式语言。
- 语法类似于EL，SpEL使用#{---}作为定界符
- SpEL为Bean的属性进行动态赋值提供了便利。





### Bean的生命周期

- Spring IOC容器可以管理Bean的生命周期，Spring允许在Bean生命周期的特定点执行定制的任务。
- 在Bean的声明里面设置init-method和destory-method属性，为Bean指定初始化和销毁方法。

```xml
&lt;bean id=&#34;car&#34; class=&#34;com.test.spring.cycle.Car&#34; init-method=&#34;init&#34; destroy-method=&#34;destory&#34;&gt;
        &lt;property name=&#34;brand&#34; value=&#34;Audi&#34;/&gt;
&lt;/bean&gt;
```





```java
public static void main(String[] args) {
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext(&#34;beans-cycle.xml&#34;);
        Car car = (Car) ctx.getBean(&#34;car&#34;);
        System.out.println(car);

        ctx.close();
    }
```



### Bean的后置处理器

- Bean的后置处理器允许在调用初始化方法前后对Bean进行额外的处理
- Bean后置处理器对IOC容器里面的所有Bean实例逐一处理。而非单一实例. 其典型应用是: 检查 Bean 属性的正确性或根据特定的标准更改 Bean 的属性。



实现**BeanPostProcessor**接口，并实现：

&gt; Object postProcessBeforeInitialization(Object bean, String beanName)	//init-method之前被调用
&gt; Object postProcessAfterInitialization(Object bean, String beanName)	//init-method之后被调用



bean：bean实例本身

beanName：IOC容器配置的Bean的名字

返回值：是实际上返回给用户的那个Bean，注意：可以在以上两个方法中修改返回的Bean，甚至返回一个新的Bean



```xml
&lt;!--配置bean的后置处理器--&gt;
    &lt;bean class=&#34;com.test.spring.cycle.MyBeanPostProcessor&#34;/&gt;
```



`MyBeanPostProcessor.java`

```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object o, String s) throws BeansException {
        System.out.println(&#34;postProcessBeforeInitialization：&#34;&#43;o&#43;&#34;,&#34;&#43;s);
        return o;
    }

    @Override
    public Object postProcessAfterInitialization(Object o, String s) throws BeansException {
        System.out.println(o&#43;s);
        return o;
    }
}
```



### 工厂方法配置Bean



#### 静态工厂方法

直接调用某一个类的静态方法就可以返回Bean实例



&gt; 通过静态工厂方法来配置Bean，不是配置静态工厂方法实例，而是配置Bean实例。





```xml
&lt;bean id=&#34;car1&#34; class=&#34;com.test.spring.factory.StaticCarFactory&#34; factory-method=&#34;getCar&#34;&gt;
        &lt;constructor-arg value=&#34;Audi&#34;/&gt;
&lt;/bean&gt;
```

class：指向静态工厂方法的全类名

factory-method：只想静态工厂方法的名字

constructor-arg：配置传入的参数



#### 实例工厂方法

现需要创建工厂本身，再调用工厂的实例方法来返回Bean实例





### FactoryBean配置

```xml
&lt;bean id=&#34;car&#34; class=&#34;com.test.spring.FactoryBean.CarFactoryBean&#34;&gt;
        &lt;property name=&#34;brand&#34; value=&#34;BMW&#34;/&gt;
&lt;/bean&gt;
```



`com.test.spring.FactoryBean.CarFactoryBean`

```java
public class CarFactoryBean implements FactoryBean {

    private String brand;

    public void setBrand(String brand) {
        this.brand = brand;
    }

    //返回bean的对象
    @Override
    public Object getObject() throws Exception {
        return new Car(brand, 400000);
    }

    //返回bean的类型
    @Override
    public Class&lt;?&gt; getObjectType() {
        return Car.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```





### 注解配置Bean



#### 在classpath中扫描组件

- Spring能够从classpath下自动扫描，侦测和实例化具有特定注解的组件



- 组件包括：
  - @Component：基本注解，标识了一个受Spring管理的组件
  - @Respository：标识持久层组件
  - @Service：标识业务层组件
  - @Controller：标识表现层组件           UserService    userService

  

- 对于扫描到的组件，Spring有默认的命名策略，使用非限定类名，第一个字母小写，**也可以在注解中通过value属性标识组件的名称**
- 在组件类上使用了特定的注解之后，还需要在Spring的配置文件中声明 `&lt;context:component-scan&gt;`
  - base-package属性指定一个需要扫描的基类包，Spring容器会将扫描的这个基类包里及其子包中的所有类
  - resource-pattern属性扫描的资源
  - `&lt;context:include-filter&gt;`子节点表示要包含的目标类
  - `&lt;context:exclude-filter&gt;`子节点表示要排除在外的目标类，需要use-default-filters配合使用

```xml
&lt;context:component-scan base-package=&#34;com.test.spring.annotation.entity&#34; resource-pattern=&#34;repository/*.class&#34;/&gt;

```



```xml
&lt;context:component-scan base-package=&#34;com.test.spring.annotation.entity&#34;&gt;
   &lt;context:exclude-filter type=&#34;annotation&#34; expression=&#34;org.springframework.stereotype.Controller&#34;/&gt;
&lt;/context:component-scan&gt;
```



`&lt;context:component-scan&gt;`元素还会自动注册AutowiredAnnotationBeanPostProcessor实例，该实例可以自动装配具有**@Autowired和@Resource，@Inject注解**的属性



#### 使用@Autowired自动装配Bean

@Autowired注解自动装配具有兼容类型的单个Bean属性

---

> Author:   
> URL: http://localhost:1313/posts/java%E5%9F%BA%E7%A1%80/spring01/  

