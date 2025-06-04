# 设计模式之结构型模式（三）


# 适配器模式
&gt; 适配器模式将某个类的接口转换成客户端期望的另一个接口表示，主要目的是兼容性，让原本因接口不匹配不能在一起工作的两个类可以协同工作，别名为包装器（Wrapper）。


## 类适配器
注意：
- 类适配器需要继承src类，这要求dst必须是接口，有一定的局限性
- src类的方法在Adapter中会被暴露出来
- Adapter需要重写src的方法


```java
/**
 * @author ：L1nker4
 * @date ： 创建于  2020/5/19 11:40
 * @description： 被适配的类
 */
public class Voltage220V {

    public int output220V(){
        int src = 220;
        System.out.println(&#34;电压：&#34;&#43; src &#43; &#34;伏&#34;);
        return src;
    }
}

/**
 * 适配接口
 */
public interface IVoltage5V {
    public int output5V();
}

/**
 * @author ：L1nker4
 * @date ： 创建于  2020/5/19 11:42
 * @description： 适配器类
 */
public class VoltageAdapter extends Voltage220V implements IVoltage5V{
    @Override
    public int output5V() {
        int src = output220V();
        return src / 44;
    }
}

/**
 * @author ：L1nker4
 * @date ： 创建于  2020/5/19 11:44
 * @description： 手机类
 */
public class Phone {

    public void charging(IVoltage5V iVoltage5V){
        if (iVoltage5V.output5V() == 5){
            System.out.println(&#34;电压为：5V，可以充电&#34;);
        }else {
            System.out.println(&#34;电压不正常，无法充电&#34;);
        }
    }
}

```

## 对象适配器
- 基本思路和类适配器模式相同，只是将Adapter类作修改，不是继承src类，而是持有src类的实例，以解决兼容性的问题，即：持有src类，实现dst接口，完成src-&gt;dst的适配。
- 根据合成复用原则，在系统中使用关联关系替代继承关系。
```java
/**
 * @author ：L1nker4
 * @date ： 创建于  2020/5/19 11:42
 * @description： 适配器类
 */
public class VoltageAdapter implements IVoltage5V {

    private Voltage220V voltage220V;

    public VoltageAdapter(Voltage220V voltage220V) {
        this.voltage220V = voltage220V;
    }

    @Override
    public int output5V() {
        if(null != voltage220V){
            int src = voltage220V.output220V();
            System.out.println(&#34;适配完成&#34;);
            return src / 44;
        }
        System.out.println(&#34;适配失败&#34;);
        return 0;
    }
}

```

## 应用
Spring MVC - HandlerAdapter
```java
public interface HandlerAdapter {
    boolean supports(Object var1);

    @Nullable
    ModelAndView handle(HttpServletRequest var1, HttpServletResponse var2, Object var3) throws Exception;

    long getLastModified(HttpServletRequest var1, Object var2);
}

```

```java
try {
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = &#34;GET&#34;.equals(method);
            if (isGet || &#34;HEAD&#34;.equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) &amp;&amp; isGet) {
                    return;
                }
            }

            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            applyDefaultViewName(processedRequest, mv);
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
```

# 桥接模式
&gt; 桥接是用于把抽象与实现解耦，使得两者可以独立变化。

该模式主要解决在多种可能会变化的情况下，用继承会造成类爆炸问题，扩展起来不灵活。

```java
/**
 * @author ：L1nker4
 * @date ： 创建于  2020/5/20 12:48
 * @description： 桥接 接口
 */
public interface DrawAPI {
    public void drawCircle(int radius, int x, int y);
}

public class RedCircle implements DrawAPI {
    @Override
    public void drawCircle(int radius, int x, int y) {
        System.out.println(&#34;Drawing Circle[ color: red, radius: &#34;
                &#43; radius &#43; &#34;, x: &#34; &#43; x &#43; &#34;, &#34; &#43; y &#43; &#34;]&#34;);
    }
}

public class GreenCircle implements DrawAPI {
    @Override
    public void drawCircle(int radius, int x, int y) {
        System.out.println(&#34;Drawing Circle[ color: green, radius: &#34;
                &#43; radius &#43; &#34;, x: &#34; &#43; x &#43; &#34;, &#34; &#43; y &#43; &#34;]&#34;);
    }
}

public abstract class Shape {

    protected DrawAPI drawAPI;
    protected Shape(DrawAPI drawAPI){
        this.drawAPI = drawAPI;
    }
    public abstract void draw();
}

public class Circle extends Shape {
    private int x, y, radius;

    public Circle(int x, int y, int radius, DrawAPI drawAPI) {
        super(drawAPI);
        this.x = x;
        this.y = y;
        this.radius = radius;
    }

    @Override
    public void draw() {
        drawAPI.drawCircle(radius, x, y);
    }
}

```

## 优点
- 抽象和实现分离
- 对客户透明，客户端不用关心细节的实现


# 装饰模式
&gt; 动态地给一个对象添加一些额外的职责，就增加功能来说，装饰模式比生成子类更为灵活。

主要有以下四个角色：
- Component：用于规范需要装饰的对象
- Concrete Component：实现Component角色，定义一个需要装饰的原始类
- Decorator：持有一个构建对象的实例，并定义一个与Component接口一致的接口
- Concrete Decorator：负责对构建对象进行装饰

```java
public interface Component {

    public void operation();
}

public class ConcreteComponent implements Component {
    @Override
    public void operation() {
        //业务
    }
}

public abstract class Decorator {

    private Component component;

    public Decorator(Component component) {
        this.component = component;
    }

    public void operation(){
        component.operation();
    }
}

public class ConcreteDecorator  extends Decorator{
    public ConcreteDecorator(Component component) {
        super(component);
    }

    public void method(){
        //自己的方法
    }

    @Override
    public void operation() {
        this.method();
        super.operation();
    }
}
```

## 优缺点
- 装饰类和被装饰类可以独立发展。
- 装饰模式是继承的一个替代方案。
- 多层装饰较为复杂。






# 组合模式
&gt; 将对象组合成树形结构以表示部分一整体的层次结构，使得用户对单个对象和组合对象的使用具有一致性。

```java

public class Employee {
   private String name;
   private String dept;
   private int salary;
   private List&lt;Employee&gt; subordinates;
 
   //构造函数
   public Employee(String name,String dept, int sal) {
      this.name = name;
      this.dept = dept;
      this.salary = sal;
      subordinates = new ArrayList&lt;Employee&gt;();
   }
 
   public void add(Employee e) {
      subordinates.add(e);
   }
 
   public void remove(Employee e) {
      subordinates.remove(e);
   }
 
   public List&lt;Employee&gt; getSubordinates(){
     return subordinates;
   }
 
   public String toString(){
      return (&#34;Employee :[ Name : &#34;&#43; name 
      &#43;&#34;, dept : &#34;&#43; dept &#43; &#34;, salary :&#34;
      &#43; salary&#43;&#34; ]&#34;);
   }   
}


public class Employee {
    private String name;
    private String dept;
    private int salary;
    private List&lt;Employee&gt; subordinates;

    public Employee(String name, String dept, int sal) {
        this.name = name;
        this.dept = dept;
        this.salary = sal;
        subordinates = new ArrayList&lt;Employee&gt;();
    }

    public void add(Employee e) {
        subordinates.add(e);
    }

    public void remove(Employee e) {
        subordinates.remove(e);
    }

    public List&lt;Employee&gt; getSubordinates() {
        return subordinates;
    }

    @Override
    public String toString() {
        return (&#34;Employee :[ Name : &#34; &#43; name
                &#43; &#34;, dept : &#34; &#43; dept &#43; &#34;, salary :&#34;
                &#43; salary &#43; &#34; ]&#34;);
    }
}


```

# 外观模式

&gt; 要求一个子系统的外部与其内部的通信必须通过一个统一的对象进行。外观模式提供一个高层次的接口，使得子系统更易使用。

```java
public interface Shape {
   void draw();
}

public class Rectangle implements Shape {
 
   @Override
   public void draw() {
      System.out.println(&#34;Rectangle::draw()&#34;);
   }
}

public class Square implements Shape {
 
   @Override
   public void draw() {
      System.out.println(&#34;Square::draw()&#34;);
   }
}

public class Circle implements Shape {
 
   @Override
   public void draw() {
      System.out.println(&#34;Circle::draw()&#34;);
   }
}

public class ShapeMaker {
   private Shape circle;
   private Shape rectangle;
   private Shape square;
 
   public ShapeMaker() {
      circle = new Circle();
      rectangle = new Rectangle();
      square = new Square();
   }
 
   public void drawCircle(){
      circle.draw();
   }
   public void drawRectangle(){
      rectangle.draw();
   }
   public void drawSquare(){
      square.draw();
   }
}
```

# 享元模式

&gt; 运用共享技术来有効地支持大量细粒度对象的复用。

- 常见于系统底层开发，能够解决重复对象的内存浪费问题。
- 经典应用场景：池技术（String常量池、数据库连接池、缓冲池）

享元模式能做到共享的关键是区分内部状态和外部状态。
- 内部状态是存储在享元对象内部的，可以共享的信息，不会随环境而变化。
- 外部状态是随环境而改变且不可以共享的状态。

```java
public interface Flyweight {

    public abstract void operation(String extrinsicState);
}

public class ConcreteFlyWeight implements Flyweight {

    //内部状态
    private String intrinsicState;

    public ConcreteFlyWeight(String intrinsicState) {
        this.intrinsicState = intrinsicState;
    }


    @Override
    public void operation(String extrinsicState) {
        System.out.println(&#34;内部状态：&#34; &#43; intrinsicState);
        System.out.println(&#34;外部状态：&#34; &#43; extrinsicState);
    }
}

public class FlyweightFactory {

    private static Map&lt;String, Flyweight&gt; pool = new HashMap&lt;&gt;();

    private FlyweightFactory() {
    }

    public static Flyweight getFlyweight(String intrinsicState){
        Flyweight flyweight = pool.get(intrinsicState);
        if(flyweight == null){
            flyweight = new ConcreteFlyWeight(intrinsicState);
            pool.put(intrinsicState, flyweight);
        }
        return flyweight;
    }
}

```

# 代理模式
&gt; 为其他对象提供一种代理以控制这个对象的访问。

该模式主要有三个角色：
- Subject（抽象主题角色）：该角色是真实主题和迪阿尼主题共同接口，以便在任何可以使用真实主题的地方都可以使用代理主题。
- Proxy Subject（代理主题角色）：也叫做代理类，该角色负责控制对真实主题的引用，负责在需要的时候创建或删除真实主题对象，并且在真实主题角色处理完毕钱后做预处理工作。
- Real Project（真实主题角色）：该角色被称为被代理角色，是业务逻辑的具体执行者。

```java
public interface Subject {

    public void request();
}

public class RealSubject implements Subject {
    @Override
    public void request() {
        //业务代码
    }
}

public class ProxySubject implements Subject {

    private Subject subject;

    public ProxySubject(Subject subject) {
        this.subject = subject;
    }

    @Override
    public void request() {
        this.beforeRequest();
        subject.request();
        this.afterRequest();
    }

    public void beforeRequest(){

    }

    public void afterRequest(){

    }
}
```

---

> Author:   
> URL: http://localhost:1313/posts/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/design-pattern-structure/  

