# 设计模式之行为型模式（四）


# 模板方法模式
&gt; 定义一个操作中的算法的框架，而将一些步骤延迟到子类中。使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

模板方法模式涉及两个角色：
- 抽象模板角色（Abstract Template）：该角色定义多个抽象操作，以便让子类实现。
- 具体模板角色（Concrete Template）：该角色实现抽象模板中的抽象方法

```java
public abstract class AbstractClass {
    protected abstract void operation();

    public void templateMethod(){
        this.operation();
    }
}

public class ConcreteClass extends AbstractClass {
    @Override
    protected void operation() {
        //业务
    }
}

public class Client {

    public static void main(String[] args) {
        AbstractClass ac = new ConcreteClass();
        ac.templateMethod();
    }
}
```

优点：
- 封装不变的部分，扩展可变部分。不变的部分封装到父类去实现，可变的通过继承进行扩展。
- 提取公共代码，便于维护，将公共部分的代码抽取出来放在父类中，维护时只需要修改父类中的代码。
- 行为由父类控制，子类实现。


# 命令模式
&gt; 将一个请求封装成一个对象，从而让你使用不同的请求把客户端参数化，对请求排队或者记录请求日志，可以提供命令的撤销和恢复功能。

该模式有四个角色：
- 命令角色：该角色声明抽象接口
- 具体命令角色：定义一个接收者和行为之间的弱耦合，实现命令方法，并调用接收者的相应操作。
- 调用者：负责调用命令对象执行请求。
- 接收者：该角色负责具体实施和执行一个请求。

```java
public interface Command {

    public void execute();
}

public class ConcreteCommand implements Command {

    private Receiver receiver;

    public ConcreteCommand(Receiver receiver) {
        this.receiver = receiver;
    }

    @Override
    public void execute() {
        this.receiver.action();
    }
}

public class Receiver {

    public void action(){
        System.out.println(&#34;执行动作&#34;);
    }
}

public class Invoker {

    private Command command;

    public void setCommand(Command command) {
        this.command = command;
    }

    public void action() {
        this.command.execute();
    }
}
```
优点：
- 类间解耦
- 可扩展性

# 责任链模式
&gt; 使多个对象都有机会处理请求，从而避免了请求的发送者和请求者之间耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有对象处理它为止。

该模式有两个角色：
- 抽象处理者角色：该角色对请求进行抽象，并定义一个方法以设定和返回对下一个处理者的引用。
- 具体处理者角色：该角色接到请求后，可以将请求处理掉，或者将请求传给下一个处理者。

```java
public abstract class Handler {

    private Handler successor;

    public abstract void handleRequest();

    public Handler getSuccessor() {
        return successor;
    }

    public void setSuccessor(Handler successor) {
        this.successor = successor;
    }
}

public class ConcreteHandler extends Handler{
    @Override
    public void handleRequest() {
        if (getSuccessor() != null){
            System.out.println(&#34;请求传递给：&#34;&#43; getSuccessor());
            getSuccessor().handleRequest();
        }else {
            System.out.println(&#34;请求处理&#34;);
        }
    }
}

```

优点：
- 将请求和处理分开
- 提高系统的灵活性

缺点：
- 链过长将降低程序的性能
- 不易调试

# 策略模式
&gt; 定义一组算法，将每个算法都封装起来，并且使它们之间可以互换。

该模式涉及三个角色：
- 环境角色：屏蔽高层模块对策略、算法的直接访问
- 抽象策略角色：对策略、算法进行抽象
- 具体策略角色：实现抽象策略中的具体操作

```java
public abstract class Strategy {

    public abstract void strategyInterface();
}

public class ConcreteStrategy extends Strategy {
    @Override
    public void strategyInterface() {
        //具体逻辑
    }
}

public class Context {

    private Strategy strategy;

    public Context(Strategy strategy) {
        this.strategy = strategy;
    }

    public void contextInterface(){
        this.strategy.strategyInterface();
    }
}
```

优点：
- 提供了管理算法族的方法
- 可以替换继承关系的办法
- 可以避免使用多重条转移语句

缺点：
- 客户端必须直到所有的策略类
- 策略模式会造成很多策略类

# 迭代器模式
&gt; 提供一种方法访问容器对象中各个元素，而又不暴露该对象的内部细节。

该模式有四个角色：
- 抽象迭代器角色：定义迭代接口
- 具体迭代器角色：负责实现迭代接口
- 抽象聚集角色：提供创建迭代器角色的接口
- 具体聚集角色：实现抽象聚集接口

```java
public interface Iterator {

    public Object next();

    public boolean hasNext();
}

public class ConcreteIterator implements Iterator {

    private ConcreteAggregate aggregate;

    private int index = 0;

    private int size = 0;

    public ConcreteIterator(ConcreteAggregate aggregate) {
        this.aggregate = aggregate;
        this.index = 0;
        this.size = aggregate.size();
    }

    @Override
    public Object next() {
        if (hasNext()) {
            return aggregate.getElement(index&#43;&#43;);
        } else {
            return null;
        }
    }

    @Override
    public boolean hasNext() {
        return index &lt; size;
    }
}

public interface Aggregate {

    public void add(Object obj);

    public Iterator createIterator();
}

public class ConcreteAggregate implements Aggregate {

    private Vector vector = new Vector();

    @Override
    public void add(Object obj) {
        this.vector.add(obj);
    }

    @Override
    public Iterator createIterator() {
        return new ConcreteIterator(this);
    }

    public int size(){
        return vector.size();
    }

    public Object getElement(int index) {
        if (index &lt; vector.size()){
            return vector.get(index);
        }else {
            return null;
        }
    }
}
```

优点：
- 简化访问容器元素的操作，具备一个同意的遍历接口
- 封装了遍历算法

# 中介者模式
&gt; 用一个中介对象封装一系列对象的交互，中介者使各对象不需要显式地相互作用，从而使其耦合松散，而且可以独立地改变它们之间地交互。

该模式主要有四个角色：
- 抽象中介者角色：定义出对象到中介者的接口
- 具体中介者角色：实现抽象中介者
- 抽象对象角色：定义出中介者到对象的接口
- 具体对象角色：实现抽象对象

```java
public abstract class Colleague {

    private Mediator mediator;

    public Colleague(Mediator mediator) {
        this.mediator = mediator;
    }

    public Mediator getMediator() {
        return mediator;
    }

    public void setMediator(Mediator mediator) {
        this.mediator = mediator;
    }

    public abstract void action();

    public void change(){
        mediator.colleagueChanged();
    }
}

public class ConcreteColleague extends Colleague {


    public ConcreteColleague(Mediator mediator) {
        super(mediator);
    }

    @Override
    public void action() {
        System.out.println(&#34;action&#34;);
    }
}

public abstract class Mediator {

    public abstract void colleagueChanged();
}

public class ConcreteMediator extends Mediator {

    private ConcreteColleague concreteColleague;


    public void createConcreteMediator(){
        concreteColleague = new ConcreteColleague(this);
    }

    @Override
    public void colleagueChanged() {
        concreteColleague.action();
    }

}
```
优点：
- 减少类间的依赖，将一对多的依赖变成一对一的依赖。
- 避免同事对象之间过度耦合
- 中介者模式将对象地行为和协作抽象化

# 观察者模式
&gt; 定义对象间一种一对多的依赖关系，使得每当一个对象改变状态，则所有依赖它的对象会得到通知并被自动更新。

该模式主要有四个角色：
- 抽象主题角色：被观察者，可以添加删除观察者角色
- 抽象观察者角色：为观察者定义接口
- 具体主题角色：将有关状态存入具体观察者对象，在内部状态改变时，通知观察者。
- 具体观察者角色：实现抽象接口。

```java
public interface Subject {

    /**
     * 添加观察者
     */
    public void attach(Observer observer);

    /**
     * 删除观察者
     */
    public void detach(Observer observer);

    /**
     * 通知观察者
     */
    public void notifyObserver();
}

public class ConcreteSubject implements Subject {

    private Vector&lt;Observer&gt; observers = new Vector&lt;&gt;();

    @Override
    public void attach(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void detach(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObserver() {
        for (Observer observer : observers){
            observer.update();
        }
    }

    public Enumeration&lt;Observer&gt; observers(){
        return observers.elements();
    }

    public void change(){
        this.notifyObserver();
    }

}

public interface Observer {

    public void update();
}

public class ConcreteObserver implements Observer {
    @Override
    public void update() {
        System.out.println(&#34;receive notice&#34;);
    }
}
```

优点：
- 两者之间是抽象耦合
- 支持向所有的观察者发出通知

缺点：
- 如果对观察者的通知是通过另外的线程进行异步投递，需要保证投递的顺序性。


# 备忘录模式
&gt; 在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态，这样，以后就可以将该对象恢复到原先保存的状态。

通俗的说，备忘录模式就是将一个对象进行备份的方法。

该模式有三个角色：
- 发起人角色：记录当前时刻的内部状态，负责定义哪种属于备份范围的状态，负责创建和恢复数据
- 备忘录角色：该角色存储发起人的内部状态
- 负责人角色：对备忘录角色进行管理、保存和提供备忘录。

```java
public class Originator {

    private String state = &#34;&#34;;

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }

    public Memento createMemento(){
        return new Memento(this.state);
    }

    public void restoreMemento(Memento memento){
        this.setState(memento.getState());
    }
}

public class Memento {

    private String state;

    public Memento(String state) {
        this.state = state;
    }

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }
}

public class Caretaker {

    private Memento memento;

    public Memento getMemento(){
        return memento;
    }

    public void setMemento(Memento memento) {
        this.memento = memento;
    }

}
```

# 访问者模式
&gt; 封装一些作用于某种数据结构中各元素的操作，它可以在不改变数据结构的前提下定义作用于这些元素的新操作。

该模式涉及五个角色：
- 抽象访问者角色：该角色声明一个或多个访问操作，定义访问者可以访问哪些元素
- 具体访问者：实现抽象访问者的接口
- 抽象元素角色：声明一个接受操作，接受一个访问者对象
- 具体元素角色：实现抽象元素中的接受接口
- 结构对象：提供遍历操作

```java
public abstract class Element {

    public abstract void accept(Visitor visitor);
}

public class ConcreteElement extends Element {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }

    public void operation(){
        System.out.println(&#34;访问元素&#34;);
    }
}

public interface Visitor {

    public void visit(ConcreteElement concreteElement);
}

public class ConcreteVisitor implements Visitor {

    @Override
    public void visit(ConcreteElement concreteElement) {
        concreteElement.operation();
    }
}
```

优点：
- 增加新的操作很容易，只需新增新的访问者类
- 访问者模式将有关的行为集中到访问者对象中

缺点：
- 增加新的元素困难，每增加一个都意味着要在抽象访问者类中新增操作
- 破坏封装
- 违背依赖倒置原则

# 状态模式
&gt; 当一个对象内在状态改变时允许改变行为，这个对象看起来像改变了其类型。

核心是封装，状态的变更引起行为的变动，外部看好像该对象对应的类发生改变。

该模式有三个角色：
- 抽象状态角色：该角色用以封装环境对象的一个状态所对应的行为。
- 具体状态角色：该角色实现抽象行为。
- 环境角色：定义客户端需要的接口，并负责具体状态的切换


```java
public abstract class State {

    private Context context;

    public void setContext(Context context) {
        this.context = context;
    }

    public abstract void handle();

}

public class ConcreteState extends State {
    @Override
    public void handle() {
        System.out.println(&#34;逻辑处理&#34;);
    }
}

public class Context {

    private static State STATE = new ConcreteState();

    private State currentState;

    public State getCurrentState() {
        return currentState;
    }

    public void setCurrentState(State currentState) {
        this.currentState = currentState;
        currentState.setContext(this);
    }

    public void handle1(){
        this.setCurrentState(STATE);
        this.currentState.handle();
    }
}
```

优点：
- 结构清晰
- 遵循设计原则
- 封装性好

# 解释器模式
&gt; 给定一门语言，定义它的文法的一种表示，并定义一个解释器，该解释器使用该表示来解释语言中的句子。

该模式有五个角色：

- 抽象表达式（Abstract Expression）角色：该角色声明一个所有的具体表达式角色都需要实现的抽象接口，该接口主要是一个解释操作interpret()方法。
- 终结符表达式（Terminal Expression）角色：该角色实现了抽象表达式角色所要求的接口，文法中的每一个终结符都有一个具体终结表达式与之对应。
- 非终结符表达式（Nonterminal Expression）角色：该角色是一个具体角色，文法中的每一条规则都对应一个非终结符表达式类。
- 环境（Context）角色：该角色提供解释器之外的一些全局信息。
- 客户端（Client）角色：该角色创建一个抽象语法树，调用解释操作。


```java
public abstract class AbstractExpression {

    public abstract Object interpreter(Context ctx);
}

public class NonterminalExpression extends AbstractExpression {

    @Override
    public Object interpreter(Context ctx) {
        return null;
    }
}

public class TerminalExpression extends AbstractExpression {
    @Override
    public Object interpreter(Context ctx) {
        return null;
    }
}

public class Context {

    private HashMap map = new HashMap();
}

public class Client {
    public static void main(String[] args) {
        Context context = new Context();
        //todo
    }
}
```

---

> Author:   
> URL: http://localhost:1313/posts/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/design-pattern-behavior/  

