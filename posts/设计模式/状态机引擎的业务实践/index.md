# 状态机引擎的业务实践




## 前言

在开发中通常会有状态属性，例如订单状态、事件处置状态、OA请假单状态等，都会根据不同的动作，去更新对应的状态。如下：

```java
int status = 0;  
if (condition1){  
    status = 1;  
}else if (condition2){  
    status = 2;  
}else if (condition3){  
    status = 3;  
}
```



上述示例存在的问题：
- 在需求增加的情况下，if.else会不断膨胀
- 代码可读性差，略微改动会导致各种问题
- 其他业务逻辑会耦合在if.else代码段中



针对系统中状态的管理，可以使用**有限状态机**去解决，有限状态机是表示有限个状态以及状态间转移的模型。状态机由事件、状态、动作三部分组成。



![image-20230220205838434](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20230220205838434.png)







## 状态机的简单实现



### 条件分支判断

最简单的方案是通过条件分支控制状态的转移，这适合于业务状态较少，并且状态跳转的逻辑较简单的场景，但是当触发事件较多时，需要嵌套多层条件判断，当需求中状态变更事件改变时，需要改动的逻辑较大。案例代码如下：

```java
public enum ActivityState {

    PREPARE(0),
    STARTED(1),
    END(2)
    ;

    public final int status;


    ActivityState(int status) {
        this.status = status;
    }

    public int getStatus() {
        return status;
    }
}

/**
 * 活动状态-状态机
 * @author l1nker4
 */
public class ActivityStateMachine {
    /**
     * 活动状态
     */
    private ActivityState currentState;

    public ActivityStateMachine() {
        this.currentState = ActivityState.PREPARE;
    }


    /**
     * 活动开始
     */
    public void begin() {
        if (currentState.equals(ActivityState.PREPARE)) {
            this.currentState = ActivityState.STARTED;
            //do something....
        }
    }

    /**
     * 活动结束
     */
    public void end(){
        if (currentState.equals(ActivityState.STARTED)){
            this.currentState = ActivityState.END;
            //do something...
        }
    }

}
```





### 状态模式实现

设计模式中的状态模式也是状态机的一种实现方式，定义了状态-行为的对应关系，并将各自的行为封装到状态类中。状态模式会引入较多的状态类和方法，需求变更时维护成本大。

```java
public interface IActivityState {

    /**
     *
     * @return
     */
    ActivityState getName();

    /**
     *
     */
    void prepareAction();

    /**
     *
     */
    void startAction();

    /**
     *
     */
    void endAction();

}

public class ActivityStateMachine {

    private IActivityState currentState;

    public ActivityStateMachine(IActivityState currentState) {
        this.currentState = new ActivityPrepareState(this);
    }
    public void setCurrentState(IActivityState currentState) {
        this.currentState = currentState;
    }

    public void prepareAction(){
        currentState.prepareAction();
    }

    public void startAction(){
        currentState.startAction();
    }

    public void endAction(){
        currentState.endAction();
    }
}

public class ActivityPrepareState implements IActivityState{

    private final ActivityStateMachine stateMachine;

    public ActivityPrepareState(ActivityStateMachine stateMachine) {
        this.stateMachine = stateMachine;
    }

    @Override
    public ActivityState getName() {
        return ActivityState.PREPARE;
    }

    @Override
    public void prepareAction() {
        stateMachine.setCurrentState(new ActivityPrepareState(stateMachine));
    }

    @Override
    public void startAction() {

    }

    @Override
    public void endAction() {

    }
}

public class ActivityStartState implements IActivityState{

    private final ActivityStateMachine stateMachine;

    public ActivityStartState(ActivityStateMachine stateMachine) {
        this.stateMachine = stateMachine;
    }

    @Override
    public ActivityState getName() {
        return ActivityState.STARTED;
    }

    @Override
    public void prepareAction() {

    }

    @Override
    public void startAction() {
        stateMachine.setCurrentState(new ActivityStartState(stateMachine));
    }

    @Override
    public void endAction() {

    }
}

public class ActivityEndState implements IActivityState {
    private final ActivityStateMachine stateMachine;

    public ActivityEndState(ActivityStateMachine stateMachine) {
        this.stateMachine = stateMachine;
    }

    @Override
    public ActivityState getName() {
        return ActivityState.END;
    }

    @Override
    public void prepareAction() {

    }

    @Override
    public void startAction() {

    }

    @Override
    public void endAction() {
        stateMachine.setCurrentState(new ActivityEndState(stateMachine));
    }
}


```



## DSL方案

DSL（[Domain-Specific Languages](https://www.martinfowler.com/dsl.html)）：针对以特定领域的语言，用于解决一类问题，例如：正则表达式可以解决字符串匹配这一类问题。可分为以下两种类型：

- Internal DSL：业务代码通过代码直接进行配置，Java Internal DSL一般直接利用建造者模式和流式接口构建
- External DSL：通过外部配置文件，运行时读取进行解析，例如XML、JSON等格式





### COLA

[COLA](https://github.com/alibaba/COLA/tree/master/cola-components/cola-component-statemachine)是阿里巴巴开源的DSL状态机实现，根据作者的介绍，相比较其他开源的状态机实现（Spring Statemachine等），其优点如下：

- 开源的状态机引擎性能较差，状态机有状态，需要考虑**线程安全**
- 开源的状态机较为复杂，太重。



#### 领域模型

COLA状态机的核心概念有：

- State：状态
- Event：状态由事件触发
- Transition：状态间转换的过程
- External Transition：stateA -&gt; stateB
- Internal Transition：steteA -&gt; stateA
- Condition：是否允许到达某种状态
- Action：state变化时执行的动作



#### demo

pom.xml：

```xml
&lt;dependency&gt;
    &lt;groupId&gt;com.alibaba.cola&lt;/groupId&gt;
    &lt;artifactId&gt;cola-component-statemachine&lt;/artifactId&gt;
    &lt;version&gt;4.3.1&lt;/version&gt;
&lt;/dependency&gt;
```



```java
public enum Events {

    STARTED(&#34;activity started&#34;),

    END(&#34;activity ended&#34;)
    ;


    private String event;

    Events(String event) {
        this.event = event;
    }

    public void setEvent(String event) {
        this.event = event;
    }
}

@Test
public void testMachine() {
        StateMachineBuilder&lt;ActivityState, Events, Object&gt; builder = StateMachineBuilderFactory.create();

        builder.externalTransition()
                .from(ActivityState.PREPARE)
                .to(ActivityState.STARTED)
                .on(Events.STARTED)
                .when(checkCondition())
                .perform(doAction());
        StateMachine&lt;ActivityState, Events, Object&gt; machine = builder.build(&#34;test&#34;);
        ActivityState activityState = machine.fireEvent(ActivityState.PREPARE, Events.STARTED, new Object());
        log.info(&#34;new state: {}&#34;, activityState);
    }


    private Condition&lt;Object&gt; checkCondition() {
        return (context -&gt; {return true;});
    }

    private Action&lt;ActivityState, Events, Object&gt; doAction() {
        return ((from, to, event, context) -&gt; {
            log.info(&#34;{} is doing, from {}, to {}, on: {}&#34;, context, from, to, event);
        });
    }

```





## Reference

[Domain-Specific Languages](https://www.martinfowler.com/dsl.html)

[实现一个状态机引擎，教你看清DSL的本质](https://blog.csdn.net/significantfrank/article/details/104996419)


---

> Author:   
> URL: http://localhost:1313/posts/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E7%8A%B6%E6%80%81%E6%9C%BA%E5%BC%95%E6%93%8E%E7%9A%84%E4%B8%9A%E5%8A%A1%E5%AE%9E%E8%B7%B5/  

