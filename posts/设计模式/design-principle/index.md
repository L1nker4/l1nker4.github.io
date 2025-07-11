# 设计模式之设计原则（一）


# 简介

## 什么是设计模式？

&gt; 每一个模式描述了一个在我们周围不断重复发生的问题，以及该问题的解决方案的核心，这样，你就能不必一次又一次地使用该方案而不必重复劳动。



## 深入理解面向对象

向下：三大面向对象机制
- 封装：隐藏内部实现
- 继承：复用现有代码
- 多态：改写对象行为

现上：深刻把握面向对象机制带来的抽象意义，理解如何使用这些机制来表达现实世界，掌握什么是“好的面向对象设计”



## 软件设计复杂的根本原因

**变化**：

- 客户需求的变化
- 技术平台的变化
- 开发团队的变化
- 市场环境的变化



## 如何解决复杂性？

分解
- 人们面对复杂性有一个常见的作法：分而治之，将大问题分解为多个小问题，将复杂问题分解为多个简单问题。

抽象
- 更高层次来讲，人们处理复杂性有一个通用的计数，即抽象。由于不能掌握全部的复杂对象，我们选择忽视它的非本质细节，而去处理泛化和理想化了的对象模型。



# 设计原则



## 单一职责原则（SRP）

&gt; 一个接口仅负责一个职责，一个类只能由一个原因引起变化。

优点：
- 类的复杂性降低，实现什么职责都有清晰明确的定义
- 可读性提高 
- 可维护性提高
- 变更引起的风险降低，变更时必不可少的，如果接口的单一职责做好，一个接口的修改只对相对应的实现类有影响。




## 里氏替换原则（LSP）
&gt; 所有引用基类的地方必须能透明的使用其子类的对象。

只要父类能出现的地方子类就可以出现，而且替换为子类也不会产生任何错误或异常。

- 子类必须完全实现父类的方法
- 子类可以有自己的方法和属性
- 覆盖或实现父类的方法时输入参数可以被放大
- 覆写或实现父类的方法时输出结果可以被缩小



## 依赖倒置原则（DIP）

&gt; - 高层模块不应该依赖低层模块，两者都应该依赖其抽象
&gt; - 抽象不应该依赖细节
&gt; - 细节应该依赖抽象



在Java语言中的表现：
- 模块间的依赖通过抽象发生，实现类之间不发生直接的依赖关系，其依赖关系是通过接口或抽象类产生的
- 接口或抽象类不依赖于实现类
- 实现类依赖接口或抽象类




对象的依赖关系有三种方式来传递：
- 构造函数传递依赖对象
- Setter方法传递依赖对象
- 接口声明依赖对象




如何实践？
- 每个类尽量都有接口或抽象类，或者抽象类与接口两者都具备
- 变量的表面类型尽量是接口或者抽象类
- 任何类都不应该从具体类派生
- 尽量不要覆写基类的方法
- 结合里氏替换原则使用：接口负责定义public属性和方法，并且声明与其他对象的依赖关系，抽象类负责公共内部构造部分的实现，实现类准确的实现业务逻辑，统统实在适当的时候对父类进行细化。



## 接口隔离原则（ISP）

&gt; - 客户端不应该依赖它不需要的接口
&gt; - 类间的依赖关系应该建立在最小的接口上



接口分为两种：
- 实例接口：在Java中声明一个类，然后用new关键字产生一个实例，它就是对一个类型的事物的描述，这是一种接口
- 类接口：interface关键字定义的接口

接口隔离原则对接口进行规范约束，包含以下四个含义：
- 接口要尽量小
- 接口要高内聚，减少对外的交互（尽量少公布public方法）
- 定制服务（只提供访问者需要的方法）
- 接口设计是有限度的（粒度越小，系统越灵活，结构越复杂，开发难度增加，可维护性降低）

如何实践？
- 一个接口只服务于一个子模块或业务逻辑
- 通过业务逻辑压缩接口中的public方法
- 已经被污染的方法，尽量去修改，若变更风险较大，则采用适配器模式进行处理
- 了解环境，深入理解业务逻辑，结合实际。



## 迪米特法则（LoD）

&gt; 一个对象应该对其他对象有最少的了解。

类和类关系越密切，耦合度越大。

也称为最少知识原则（LKP），通俗的讲：一个对象应该对自己需要耦合或者调用的类知道的最少，你的内部如何复杂和我都没有关系，我就知道你提供这么多public方法，我就调用这么多，其他我一概不关心。

迪米特法则对类的低耦合提出了明确的要求，其包含以下四层含义：
- 只和朋友交流（朋友类：出现在成员变量，方法输入输出参数的类）。
- 朋友之间有距离的。
- 是自己的就是自己的（一个方法放在本类种，既不增加类间关系，也对本类不产生负面影响，那就放在本类之中）。
- 谨慎使用Serializable。



迪米特法则的核心观念就是类间解耦，弱耦合，只有弱耦合之后，类的复用率才可以提高，要求的结果就是产生了大量中转或跳转类，导致系统复杂性提高，同时也为维护带来了难度。



## 开闭原则（OCP）

&gt; 一个软件实体如类，模块和函数应该对扩展开放，对修改关闭。

- 对扩展开放（提供方），对修改关闭（使用方）
- 用抽象构建框架，用实现扩展细节
- 一个软件实体应该通过扩展来实现变化，而不是通过修改已有的代码来实现变化。
- 遵循的所有原则，以及设计模式的目的就是遵循开闭原则。

---

> Author:   
> URL: http://localhost:1313/posts/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/design-principle/  

