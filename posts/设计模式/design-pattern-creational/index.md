# 设计模式之创建型模式（二）


# 单例模式

采取一定的方法保证在整个软件系统中，对某个类只能存在一个对象实例，并且该类只提供一个取得其对象实例的方法（静态方法）。

比如Hibernate的SessionFactory，它充当数据存储源的代理，并负责创建Session对象。

## 饿汉式

### 静态常量方法

demo：
```java
class Singleton{

    /**
     * 构造器私有化，外部不能new
     */
    private Singleton(){

    }

    private final static Singleton INSTANCE = new Singleton();

    public static Singleton getInstance(){
        return INSTANCE;
    }
}
```

- 优点
    - 写法简单，在类装载的时候就完成实例化，避免了线程安全问题。
- 缺点
    - 没有达到`lazy loading`的效果，如果一直没有使用，就会造成内存浪费。

### 静态代码块

demo：
```java
class Singleton{

    /**
     * 构造器私有化，外部不能new
     */
    private Singleton(){
        INSTANCE = new Singleton();
    }

    private static Singleton INSTANCE ;

    public static Singleton getInstance(){
        return INSTANCE;
    }
}
```

## 懒汉式

### 线程不安全

demo：

```java
class Singleton{

    /**
     * 构造器私有化，外部不能new
     */
    private Singleton(){
    }

    private static Singleton INSTANCE ;

    public static Singleton getInstance(){
        if (INSTANCE == null){
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }
}
```
- 优点
    - 起到了`lazy loading`的效果，当时只能在单线程下使用
- 缺点
    - 多线程下会出现线程安全问题。 

### 线程安全
加上`synchronized`关键字。

```java
class Singleton{

    /**
     * 构造器私有化，外部不能new
     */
    private Singleton(){
    }

    private static Singleton INSTANCE ;

    public static synchronized Singleton getInstance(){
        if (INSTANCE == null){
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }
}
```
- 缺点
    - 效率低

### 静态内部类

demo：
```java
class Singleton{

    /**
     * 构造器私有化，外部不能new
     */
    private Singleton(){
    }

    private static class SingletonInstance{
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance(){
        return SingletonInstance.INSTANCE;
    }
}
```
- 优点
    - 避免线程不安全，利用静态内部类特点实现延迟加载，效率高。 

## 单例模式的优点
- 单例模式在内存中只有一个实例，减少了内存开支，特别是一个对象频繁的创建，销毁时。
- 由于单例模式只生成一个实例，所以减少了系统的性能开销。
- 单例模式可以避免对资源的多重占用，例如写文件操作，避免对同一个资源文件同时操作。
- 单例模式可以在系统设置全局的访问点，优化和资源访问，例如可以设计一个单例类，负责所有数据表的映射处理。


## 单例模式的缺点
- 单例模式一般没有接口，扩展困难
- 单例模式对测试是不利的，没有接口不能使用mock的方式虚拟一个对象
- 单例模式与单一职责原则有冲突，一个类只实现一个逻辑


## 单例模式的使用场景

- 要求生成唯一序列号的环境
- 在整个项目需要一个共享访问点或共享数据，例如一个Web页面上的计数器，使用单例模式保持计数器的值，并确保是线程安全的。
- 创建一个对象需要消耗的资源过多，如要访问IO和数据库等资源
- 需要定义大量的静态变量和静态方法的环境



# 工厂模式



## 简单工厂模式(静态工厂模式)

简单工厂模式是有一个工厂对象决定创建出哪一种产品类的实例。

```java
public abstract class Product {
    public void method() {
        //业务逻辑
    }
    
    public abstract void method2();
}


public class Product1 extends Product {
    @Override
    public void method2() {

    }
}

/**
 * @author ：L1nker4
 * @date ： 创建于  2020/5/15 17:25
 * @description： 简单工厂模式
 */
public class SimpleFactory {

    public  &lt;T extends Product&gt; T createProduct(Class&lt;T&gt; c) {
        Product product = null;
        try {
            product = (Product) Class.forName(c.getName()).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return (T) product;
    }
}
```



## 工厂方法模式

&gt; 定义一个用于创建对象的接口，让子类决定实例化哪一个类，工厂方法使一个类的实例化延迟到其子类。

在工厂方法模式中，抽象产品类Product负责定义产品的共性，实现对事物最抽象的定义，Creator为抽象创建类，也就是抽象工厂，具体如何创建产品类是由具体的实现共产ConcreteCreator完成的。

```java
public abstract class Creator {
    
    public abstract &lt;T extends Product&gt; T createProduct(Class&lt;T&gt; c);
}

public class ConcreteCreator extends Creator {
    @Override
    public &lt;T extends Product&gt; T createProduct(Class&lt;T&gt; c) {
        Product product = null;
        try {
            product = (Product) Class.forName(c.getName()).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return (T) product;
    }
}
```



## 抽象工厂模式

&gt; 为创建一组相关或相互依赖的对象提供一个接口，而且无需指定他们的具体类。

- 抽象工厂模式可以将简单工厂模式和工厂方法模式进行整合。
- 从设计层面看，抽象工厂模式就是对简单工厂模式的改进。

```java
public abstract class AbstractProductA {
    
    //每个产品共有的方法
    public void shareMethod() {
        
    }
    //每个产品相同方法，不同实现
    public abstract void doSomething();
}


public class ProductA extends AbstractProductA{
    @Override
    public void doSomething() {
        System.out.println(&#34;产品A1的实现方法&#34;);
    }
}


public abstract class AbstractCreator {
    
    //创建A产品
    public abstract AbstractProductA createProductA();
}


public class Creator extends AbstractCreator{
    @Override
    public AbstractProductA createProductA() {
        return new ProductA();
    }
}

```



## 应用

java.util.Calendar

```java
private static Calendar createCalendar(TimeZone zone,
                                           Locale aLocale)
    {
        CalendarProvider provider =
            LocaleProviderAdapter.getAdapter(CalendarProvider.class, aLocale)
                                 .getCalendarProvider();
        if (provider != null) {
            try {
                return provider.getInstance(zone, aLocale);
            } catch (IllegalArgumentException iae) {
                // fall back to the default instantiation
            }
        }

        Calendar cal = null;

        if (aLocale.hasExtensions()) {
            String caltype = aLocale.getUnicodeLocaleType(&#34;ca&#34;);
            if (caltype != null) {
                switch (caltype) {
                case &#34;buddhist&#34;:
                cal = new BuddhistCalendar(zone, aLocale);
                    break;
                case &#34;japanese&#34;:
                    cal = new JapaneseImperialCalendar(zone, aLocale);
                    break;
                case &#34;gregory&#34;:
                    cal = new GregorianCalendar(zone, aLocale);
                    break;
                }
            }
        }
        if (cal == null) {
            // If no known calendar type is explicitly specified,
            // perform the traditional way to create a Calendar:
            // create a BuddhistCalendar for th_TH locale,
            // a JapaneseImperialCalendar for ja_JP_JP locale, or
            // a GregorianCalendar for any other locales.
            // NOTE: The language, country and variant strings are interned.
            if (aLocale.getLanguage() == &#34;th&#34; &amp;&amp; aLocale.getCountry() == &#34;TH&#34;) {
                cal = new BuddhistCalendar(zone, aLocale);
            } else if (aLocale.getVariant() == &#34;JP&#34; &amp;&amp; aLocale.getLanguage() == &#34;ja&#34;
                       &amp;&amp; aLocale.getCountry() == &#34;JP&#34;) {
                cal = new JapaneseImperialCalendar(zone, aLocale);
            } else {
                cal = new GregorianCalendar(zone, aLocale);
            }
        }
        return cal;
    }
```



# 原型模式

使用原型实例指定创建对象的种类，并且通过拷贝这些原型，创建新的对象。

- 创建新的对象比较复杂时，可以利用原型模式简化对象的创建过程，同时提高效率。
- 不用重新初始化对象，而是动态地获得对象运行时的状态。
- 需要注意深拷贝和浅拷贝。

```java
public class Sheep implements Cloneable {

    private String name;
    private int age;
    private String color;
    
    @Override
    protected Object clone() throws CloneNotSupportedException {
        Sheep sheep = null;
        try {
            sheep = (Sheep) super.clone();
        }catch (Exception e){
            System.out.println(e.getMessage());
        }
        return sheep;
    }
}

public class Main {
    public static void main(String[] args) throws CloneNotSupportedException {
        Sheep sheep = new Sheep(&#34;tom&#34;, 1, &#34;white&#34;);
        Sheep sheep1 = (Sheep) sheep.clone();
    }
}
```

## 应用
Spring---AbstractBeanFactory

```java
else if (mbd.isPrototype()) {
					// It&#39;s a prototype -&gt; create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

```

# 建造者模式
可以将复杂对象的构造过程抽象出来，使这个抽象过程的不同实现方法可以构造出不同表现的对象。

四个角色：
- Product：具体的产品对象
- Builder：创建一个Product的各个部件的接口
- ConcreteBuilder：实现接口，构建和装配各个部件
- Director：构建一个使用Builder接口的对象，主要用于创建一个复杂的对象，主要两个作用：
    - 隔离了客户与对象的生产过程
    - 负责控制产品对象的生产过程


```java

public class House {

    private String baise;

    private String wall;

    private String roofed;
}


public abstract class HouseBuilder {

    protected House house = new House();

    public abstract void buildBasic();
    public abstract void buildWall();
    public abstract void roofed();

    public House buildHouse() {
        return house;
    }
}

public class CommonHouse extends HouseBuilder{
    @Override
    public void buildBasic() {
        System.out.println(&#34;地基&#34;);
    }

    @Override
    public void buildWall() {
        System.out.println(&#34;砌墙&#34;);
    }

    @Override
    public void roofed() {
        System.out.println(&#34;封顶&#34;);
    }
}

public class HouseDirector {

    HouseBuilder houseBuilder = null;

    public HouseDirector(HouseBuilder houseBuilder){
        this.houseBuilder = houseBuilder;
    }

    public void setHouseBuilder(HouseBuilder houseBuilder) {
        this.houseBuilder = houseBuilder;
    }

    //如何处理建房子的过程，交给指挥者
    public House constructHouse(){
        houseBuilder.buildBasic();
        houseBuilder.buildWall();
        houseBuilder.roofed();
        return houseBuilder.buildHouse();
    }
}
```

## 应用

StringBuilder：

```
Appendable接口定义了多个append方法（抽象方法）是抽象建造者。
AbstractStringBuilder实现了Appendable接口方法，该类已经是建造者，只是不能实例化
StringBuilder既充当了指挥者角色，同时充当了具体的建造者。
```

---

> Author:   
> URL: http://localhost:1313/posts/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/design-pattern-creational/  

