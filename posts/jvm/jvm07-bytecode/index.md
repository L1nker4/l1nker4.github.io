# JVM之字节码指令（七）


## 字节码指令

Java虚拟机的指令是由一个字节长度的，代表某种特定操作含义的数字（称为操作码，Opcode），以及跟随其后的零至多个代表此操作的参数，称为操作数（Operand）构成。由于Java虚拟机面向操作数栈而不是寄存器的架构，所以大多数指令都不包含操作数，只有一个操作码。指令参数存放在操作数栈中。

由于Java虚拟机操作码的长度为一个字节（0-255），这意味着指令集的操作码总数不能超过256条。


### 加载和存储指令

- 将一个局部变量加载到操作数栈：iload、iload\_&lt;n&gt;、lload、lload\_&lt;n&gt;、fload、fload\_&lt;n&gt;、dload、dload\_&lt;n&gt;、aload、aload_&lt;n&gt;
- 将一个数值从操作数栈存储到局部变量表：istore、istore\_&lt;n&gt;、lstore、lstore\_&lt;n&gt;、fstore、fstore\_&lt;n&gt;、dstore、dstore\_&lt;n&gt;、astore、astore\_&lt;n&gt;
- 将一个常量加载到操作数栈：bipush、sipush、ldc、ldc\_w、ldc2\_w、aconst\_null、iconst\_m1、iconst\_&lt;i&gt;
- 扩充局部变量表的访问索引的指令

&lt;br&gt;

### 运算指令

- 加法指令：iadd、ladd、fadd、dadd
- 减法指令：isub、lsub、fsub、dsub
- 乘法指令：imul、lmul、fmul、dmul
- 除法指令：idiv、ldiv、fdiv、ddiv
- 求余指令：irem、lrem、frem、drem
- 取反指令：ineg、lneg、fneg、dneg
- 位移指令：ishl、ishr、iushr、lshl、lshr、lushr
- 按位或指令：ior、lor
- 按位与指令：iand、land
- 按位异或指令：ixor、lxor
- 局部变量自增指令：iinc
- 比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp

&lt;br&gt;

### 类型转换指令
JVM直接支持小范围类型向大范围类型的安全转换：
- int -&gt; long/float/double
- long -&gt; float/double
- float -&gt; double

处理窄化类型转换时，要用转换指令来完成。包括i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l、d2f.

在将int或long类型窄化转换成整数类型T的时候，转换过程仅仅是简单丢弃除最低位N字节以外的内容，N是类型T的数据类型长度。这将可能导致转换结果与输入值的正负号不同。

&lt;br&gt;

### 对象创建与访问指令

虽然类实例和数组都是对象，但是JVM对类实例和数组的创建使用不同的字节码指令，对象创建后，可以通过对象访问指令获取对象实例或者数组实例中的字段或者数组元素，包括：
- 创建类实例指令：new
- 创建数组的指令：newarray，anewarray，multianewarray
- 访问类字段（static字段）和实例字段（非static字段）的指令：getfield、putfield、getstatic、putstatic
- 把一个数组元素加载到操作数栈的指令：baload、caload、saload、iaload、laload、faload、daload、aaload
- 将一个操作数栈的值存储到数组元素中的指令：bastore、castore、sastore、iastore、fastore、dastore、aastore
- 取数组长度的指令：arraylength
- 检查类实例类型的指令：instanceof、checkcast

&lt;br&gt;

### 操作数栈管理指令

- 将操作数栈的栈顶一个或者两个元素出栈：pop、pop2
- 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1、dup2_x1、dup_x2、dup2_x2
- 将栈最顶端的两个数值交换：swap

&lt;br&gt;

### 控制转移指令
可以认为控制指令就是在有条件或者无条件地修改PC寄存器地值，控制指令包括：

- 条件分支：ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq、if_acmpne
- 复合条件分支：tableswitch、lookupswitch
- 无条件分支：goto、goto_w、jsr、jsr_w，ret

&lt;br&gt;

### 方法调用和返回指令


方法调用指令包括：
- invokevirtual：用于调用对象地实例方法，根据对象地实际类型进行分派
- invokeinterface：用于调用接口方法，他会在运行时搜索一个实现了这个接口地方法地对象，找出合适地方法进行调用
- invokespecial：用于调用一些需要特殊处理的实例方法，包括实例初始化方法，私有方法和父类方法
- invokestatic：用于调用类静态方法
- invokedynamic：用于在运行时动态解析出调用点限定符所引用的方法，并执行该方法。

方法返回指令包括
- ireturn（返回值为Boolean，char，short，int时使用）
- lreturn
- freturn
- dreturn
- areturn
- return（void）

&lt;br&gt;

### 异常处理指令
Java程序中显式抛出异常的操作（throw）都由athrow指令来实现。JVM规范规定许多运行时异常会在其他Java虚拟机指令检测到异常状态时自动抛出。
处理异常不是由字节码实现的，而是采用异常表实现的。



### 同步指令

Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor，也被称为锁）来实现的。

方法的同步是隐式的，无须通过字节码指令来控制，它实现在方法调用和返回操作中。JVM可以从方法常量池的方法表结构中`ACC_SYNCHRONIZED`访问标志得知一个方法是否被声明为同步方法，当方法调用时，调用指令将会检查方法的该标志是否被设置，如果设置，执行线程就要求先成功持有管程，然后才能执行方法。最后当方法完成时释放管程。



同步一段指令集通常时由Java语言中的synchronized语句块来表示的，JVM指令集中由monitorenter和monitorexit两条指令来支持synchronized关键字的语义。

---

> Author:   
> URL: http://localhost:1313/posts/jvm/jvm07-bytecode/  

