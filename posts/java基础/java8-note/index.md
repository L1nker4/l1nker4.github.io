# Java8特性


## Lambda表达式


### 函数式编程简介

&gt; 函数式编程（英语：functional programming）或称函数程序设计、泛函编程，是一种编程范式，它将电脑运算视为函数运算，并且避免使用程序状态以及易变对象。其中，λ演算（lambda calculus）为该语言最重要的基础。而且，λ演算的函数可以接受函数当作输入（引数）和输出（传出值）。（摘自Wikipedia）

面向对象编程是对数据进行抽象；函数式编程是对行为进行抽象。

核心思想: 使用不可变值和函数，函数对一个值进行处理，映射成另一个值。



### Lambda语法格式

```java
		//无参数，无返回值
        Runnable runnable = ()-&gt; System.out.println(&#34;Hello world&#34;);
        runnable.run();

        //有一个参数，无返回值
        Consumer&lt;String&gt; consumer = (e) -&gt; System.out.println(e);
        consumer.accept(&#34;Hello world&#34;);

        //两个参数，多条语句
        Comparator&lt;Integer&gt; comparator = (a, b) -&gt; {
            System.out.println(&#34;Hello world&#34;);
            return Integer.compare(a, b);
        };

        //lambda中只有一条语句，可简写
        Comparator&lt;Integer&gt; comparator1 = (a, b) -&gt; Integer.compare(a, b);

        //省略参数类型
        Comparator&lt;Integer&gt; comparator2 = (Integer a, Integer b) -&gt; Integer.compare(a, b);
        Comparator&lt;Integer&gt; comparator3 = (a, b) -&gt; Integer.compare(a, b);
```



### 函数式接口

接口中只有一个抽象方法的接口（不包括默认方法），称为函数式接口。使用`@FunctionInterface`注解修饰，用来检查是否满足条件。



|   函数式接口   | 参数类型 | 返回类型 |                      用途                      |
| :------------: | :------: | :------: | :--------------------------------------------: |
|  Cunsumer&lt;T&gt;   |    T     |   void   |     对类型为T的对象操作`void accept(T t)`      |
|  Supplier&lt;T&gt;   |    无    |    T     |           返回类型为T的对象`T get()`           |
| Function&lt;T, R&gt; |    T     |    R     | 对类型T的对象操作，返回R类型对象`R apply(T t)` |
|  Predicate&lt;T&gt;  |    T     | boolean  |  确定类型T的对象，是否满足`boolean test(T t)`  |



## Stream



### 概念

&gt; Java 8 中的 Stream 是对集合（Collection）对象功能的增强，它专注于对集合对象进行各种非常便利、高效的聚合操作（aggregate operation），或者大批量数据操作 (bulk data operation)。



### API



#### 筛选和切片

|        方法         |                            描述                            |
| :-----------------: | :--------------------------------------------------------: |
| filter(Predicate p) |               接收Lambda，从流中排除某些元素               |
|     distinct()      |        筛选，通过`hashCode()`和`equals`去除重复元素        |
| limit(long maxSize) |                 使得集合元素不超过给定数量                 |
|    skip(long n)     | 跳过元素，返回一个去掉了前n个元素的流，不足n个，则返回空流 |



#### 排序

|                   方法                   |                描述                |
| :--------------------------------------: | :--------------------------------: |
|                 sorted()                 | 产生一个新流，其中按照自然顺序排序 |
| sorted(Comparator&lt;? super T&gt; comparator) |         按照comparator排序         |



#### 映射

|                 方法                  |                             描述                             |
| :-----------------------------------: | :----------------------------------------------------------: |
|         map(Function mapper)          |   接收函数，该函数应用到每个元素，并将其映射为一个新的元素   |
| mapToDouble(ToDoubleFunction mapper); | 接收函数，该函数应用到每个元素，并将其映射为一个新的DoubleStream |
|   mapToLong(ToLongFunction mapper)    | 接收函数，该函数应用到每个元素，并将其映射为一个新的LongStream |
|    mapToInt(ToIntFunction mapper)     | 接收函数，该函数应用到每个元素，并将其映射为一个新的IntStream |
|          flatMap(Function f)          | 接收函数，该函数应用到每个元素，并将流中的每个值换成另一个流，然后将所有流连成一个流 |



#### 查找和匹配

|            方法            |           描述           |
| :------------------------: | :----------------------: |
|   allMatch(Predicate p)    |   检查是否匹配所有元素   |
|   anyMatch(Predicate p)    | 检查是否至少匹配一个元素 |
|   noneMatch(Predicate p)   | 检查是否没有匹配所有元素 |
|        findFirst()         |      返回第一个元素      |
|         findAny()          |   返回当前流中任意元素   |
|          count()           |     返回流中元素总数     |
| max(Comparator comparator) |      返回流中最大值      |
| min(Comparator comparator) |      返回流中最小值      |



#### 收集

|         方法         |                      描述                       |
| :------------------: | :---------------------------------------------: |
| collect(Collector c) | 将流转换为其他形式，接收一个Collector接口的实现 |



#### Demo

```java
public static void main(String[] args) {
        employeeList.stream()
                .filter((x)-&gt; x.getAge() &gt; 20)
                .forEach(System.out::println);
        System.out.println(&#34;----------------&#34;);
        employeeList.stream()
                .distinct()
                .forEach(System.out::println);

        List&lt;String&gt; stringList = Arrays.asList(&#34;aaa&#34;, &#34;bbb&#34;, &#34;ccc&#34;, &#34;ddd&#34;, &#34;eee&#34;);
        stringList.stream()
                .map(String::toUpperCase)
                .forEach(System.out::println);
        System.out.println(&#34;-------------------&#34;);
        employeeList.stream()
                .map(Employee::getName)
                .forEach(System.out::println);
    }
```



### 并行流

并行流将一个内容分成多个数据块，并用不同的线程分别处理每个数据块的流。在`Stream API`中可以声明性地通过`parallel()` 与`sequential()`在并行流与顺序流之间进行切换。

`Stream`中的并行流底层使用`Fork/Join`框架。



## Optional



### 概念

&gt; A container object which may or may not contain a non-null value.If a value is present, isPresent() will return true and get() will return the value.



### API



#### of

&gt; 为非null的值创建一个Optional。

传入null，则会抛出`NullPointerException `。

```java
public static &lt;T&gt; Optional&lt;T&gt; of(T value) {
        return new Optional&lt;&gt;(value);
}
```



#### ofNullable

&gt; 为指定的值创建一个Optional，如果指定的值为null，则返回一个空的Optional。

```java
public static &lt;T&gt; Optional&lt;T&gt; ofNullable(T value) {
        return value == null ? empty() : of(value);
}

public static&lt;T&gt; Optional&lt;T&gt; empty() {
        Optional&lt;T&gt; t = (Optional&lt;T&gt;) EMPTY;
        return t;
}
```



#### isPresent

&gt; 如果值存在返回true，否则返回false。

```java
public boolean isPresent() {
        return value != null;
}
```



#### get

&gt; 如果Optional有值则将其返回，否则抛出NoSuchElementException。

```java
public T get() {
        if (value == null) {
            throw new NoSuchElementException(&#34;No value present&#34;);
        }
        return value;
}
```



#### ifPresent

&gt; 如果Optional实例有值则为其调用consumer，否则不做处理

```java
public void ifPresent(Consumer&lt;? super T&gt; consumer) {
        if (value != null)
            consumer.accept(value);
}
```



#### orElse

&gt; 如果有值则将其返回，否则返回指定的其它值。

```java
public T orElse(T other) {
        return value != null ? value : other;
}
```



#### orElseGet

&gt; orElseGet方法可以接受Supplier接口的实现用来生成默认值。

```java
public T orElseGet(Supplier&lt;? extends T&gt; other) {
        return value != null ? value : other.get();
}
```



#### orElseThrow

&gt; 如果有值则将其返回，否则抛出supplier接口创建的异常。

```java
public &lt;X extends Throwable&gt; T orElseThrow(Supplier&lt;? extends X&gt; exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
}
```



#### map

&gt; 为空则返回空Optional，不为空返回mapper.apply后的数据

```java
public&lt;U&gt; Optional&lt;U&gt; map(Function&lt;? super T, ? extends U&gt; mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Optional.ofNullable(mapper.apply(value));
        }
}
```



#### flatMap

&gt; 与map类似，返回值不用Optional封装

```java
public&lt;U&gt; Optional&lt;U&gt; flatMap(Function&lt;? super T, Optional&lt;U&gt;&gt; mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Objects.requireNonNull(mapper.apply(value));
        }
}
```



#### filter

&gt; 如果有值并且满足断言条件则返回封装该值的Optional，否则返回空Optional。

```java
public Optional&lt;T&gt; filter(Predicate&lt;? super T&gt; predicate) {
        Objects.requireNonNull(predicate);
        if (!isPresent())
            return this;
        else
            return predicate.test(value) ? this : empty();
}
```



## 默认方法



### 概念

&gt; 接口可以有实现方法，而且不需要实现类去实现其方法。只需在方法名前面加个default关键字即可。



```java
public interface A {
    default void test(){
       System.out.println(&#34;hello world&#34;);
    }
}
```



若一个接口中定义了一个默认方法，而另外一个父类或接口中又定义了一个同名的方法时：

- 选择父类中的方法。如果一个父类提供了具体的实现，那么接口中具有相同名称和参数的默认方法会被忽略。
- 接口冲突。如果一个父接口提供一个默认方法，而另一个接口也提供了一个具有相同名称和参数列表的方法(不管方法是否是默认方法)，那么必须覆盖该方法来解决冲突。

```java
public interface A{

    default String print(){
        return &#34;A&#34;;
    }
}

public class B implements A {

    @Override
    public String print(){
        return &#34;B&#34;;
    }
}

public static void main(String[] args) {
        A a = new B();
        System.out.println(a.print());      //B
}
```



## 静态方法

接口中允许存在静态方法。

```java
public interface A{
    static void print(){
        System.out.println(&#34;hello&#34;);
    }
}
```





## 时间API



Demo：

```java
public static void main(String[] args) {
        LocalDate today = LocalDate.now();
        // 今天加一天
        LocalDate tomorrow = today.plus(1, ChronoUnit.DAYS);
        // 明天减两天
        LocalDate yesterday = tomorrow.minusDays(2);

        // 2014 年七月的第四天
        LocalDate independenceDay = LocalDate.of(2014, Month.JULY, 4);
        DayOfWeek dayOfWeek = independenceDay.getDayOfWeek();
        System.out.println(dayOfWeek);

        DateTimeFormatter germanFormatter =
                DateTimeFormatter
                        .ofLocalizedDate(FormatStyle.MEDIUM)
                        .withLocale(Locale.GERMAN);

        LocalDate xmas = LocalDate.parse(&#34;24.12.2014&#34;, germanFormatter);
        System.out.println(xmas);

        LocalDateTime sylvester = LocalDateTime.of(2014, Month.DECEMBER, 31, 23, 59, 59);

        DayOfWeek dayOfWeek1 = sylvester.getDayOfWeek();
        System.out.println(dayOfWeek1);

        Month month = sylvester.getMonth();
        System.out.println(month);

    // 获取改时间是该天中的第几分钟
        long minuteOfDay = sylvester.getLong(ChronoField.MINUTE_OF_DAY);
        System.out.println(minuteOfDay);

        DateTimeFormatter formatter =
                DateTimeFormatter
                        .ofPattern(&#34;MMM dd, yyyy - HH:mm&#34;);

        LocalDateTime parsed = LocalDateTime.parse(&#34;Nov 03, 2014 - 07:13&#34;, formatter);
        String string = formatter.format(parsed);
        System.out.println(string);


    }
```



---

> Author:   
> URL: http://localhost:1313/posts/java%E5%9F%BA%E7%A1%80/java8-note/  

