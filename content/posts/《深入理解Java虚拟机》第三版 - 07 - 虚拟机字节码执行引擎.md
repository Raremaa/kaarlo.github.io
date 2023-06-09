---
title: "《深入理解Java虚拟机》第三版 - 07 - 虚拟机字节码执行引擎"
date: 2022-09-08 19:44:43.38
draft: false
type: "post"
showTableOfContents: true
tags: ["Java","虚拟机"]
---

# 1. 运行时栈帧结构

**Java虚拟机以方法作为最基本的执行单元，“栈帧”（Stack Frame）则是用于支持虚拟机进行方法调用和方法执行背后的数据结构，它也是虚拟机运行时数据区中的虚拟机栈（Virtual Machine Stack）的栈元素。**

**栈帧包括了局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息，每一个方法从调用开始至执行结束的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程**。

**在编译Java程序源码的时候，栈帧中需要多大的局部变量表，需要多深的操作数栈就已经被分析计算出来，并且写入到方法表的Code属性之中。换言之，一个栈帧需要分配多少内存，并不会受到程序运行期变量数据的影响，而仅仅取决于程序源码和具体的虚拟机实现的栈内存布局形式。**

**一个线程中的方法调用链可能会很长，以Java程序的角度来看，同一时刻、同一条线程里面，在调用堆栈的所有方法都同时处于执行状态。而对于执行引擎来讲，在活动线程中，只有位于栈顶的方法才是在运行的，只有位于栈顶的栈帧才是生效的，其被称为“当前栈帧”（Current Stack Frame），与这个栈帧所关联的方法被称为“当前方法”（Current Method）。执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作。**

![Untitled](https://img.masaiqi.com/202209081944288.png)

## 1.1. 局部变量表

**局部变量表（Local Variables Table）是一组变量值的存储空间，用于存放方法参数和方法内部定义的局部变量。在Java程序被编译为Class文件时，就在方法的Code属性的max_locals数据项中确定了该方法所需分配的局部变量表的最大容量。**

**局部变量表的容量以变量槽（Variable Slot）为最小单位**，《Java虚拟机规范》中并没有明确指出一个变量槽应占用的内存空间大小，只是很有导向性地说到每个变量槽都应该能存放一个boolean、byte、char、short、int、float、reference或returnAddress类型的数据，这8种数据类型，都可以使用32位或更小的物理内存来存储，但这种描述与明确指出“每个变量槽应占用32位长度的内存空间”是有本质差别的，**它允许变量槽的长度可以随着处理器、操作系统或虚拟机实现的不同而发生变化，保证了即使在64位虚拟机中使用了64位的物理内存空间去实现一个变量槽，虚拟机仍要使用对齐和补白的手段让变量槽在外观上看起来与32位虚拟机中的一致。**

**对于64位的数据类型，Java虚拟机会以高位对齐的方式为其分配两个连续的变量槽空间。Java语言中明确的64位的数据类型只有long和double两种**。**这里把long和double数据类型分割存储的做法与“long和double的非原子性协定”中允许把一次long和double数据类型读写分割为两次32位读写的做法有些类似，不过，由于局部变量表是建立在线程堆栈中的，属于线程私有的数据，无论读写两个连续的变量槽是否为原子操作，都不会引起数据竞争和线程安全问题。**

Java虚拟机通过索引定位的方式使用局部变量表，索引值的范围是从0开始至局部变量表最大的变量槽数量。如果访问的是32位数据类型的变量，索引N就代表了使用第N个变量槽，如果访问的是64位数据类型的变量，则说明会同时使用第N和N+1两个变量槽。对于两个相邻的共同存放一个64位数据的两个变量槽，虚拟机不允许采用任何方式单独访问其中的某一个，《Java虚拟机规范》中明确要求了如果遇到进行这种操作的字节码序列，虚拟机就应该在类加载的校验阶段中抛出异常。

当一个方法被调用时，Java虚拟机会使用局部变量表来完成参数值到参数变量列表的传递过程，即实参到形参的传递。如果执行的是实例方法（没有被static修饰的方法），那局部变量表中第0位索引的变量槽默认是用于传递方法所属对象实例的引用，在方法中可以通过关键字“this”来访问到这个隐含的参数。其余参数则按照参数表顺序排列，占用从1开始的局部变量槽，参数表分配完毕后，再根据方法体内部定义的变量顺序和作用域分配其余的变量槽。

**为了尽可能节省栈帧耗用的内存空间，局部变量表中的变量槽是可以重用的，方法体中定义的变量，其作用域并不一定会覆盖整个方法体。如果当前字节码PC计数器的值已经超出了某个变量的作用域，那这个变量对应的变量槽就可以交给其他变量来重用。不过，这样的设计除了节省栈帧空间以外，还会伴随有少量额外的副作用**，例如在某些情况下变量槽的复用会直接影响到系统的垃圾收集行为。

**如果遇到一个方法，其后面的代码有一些耗时很长的操作，而前面又定义了占用了大量内存但实际上已经不会再使用的变量，手动将其设置为null值是一个“奇技”**。原因在于如果在前面定义了一个占大量内存的变量，后续没有任何操作局部变量表的场景时，此时大内存变量并没有被回收（因为JVM判断有可能后续被复用），不过：

- **控制变量作用域更好**
- **在实际情况中，即时编译才是虚拟机执行代码的主要方式，赋null值的操作在经过即时编译优化后几乎是一定会被当作无效操作消除掉的，这时候将变量设置为null就是毫无意义的行为。**

## 1.2. 操作数栈

**操作数栈（Operand Stack）也常被称为操作栈，它是一个后入先出（Last In First Out，LIFO）栈。同局部变量表一样，操作数栈的最大深度也在编译的时候被写入到Code属性的max_stacks数据项之中。操作数栈的每一个元素都可以是包括long和double在内的任意Java数据类型。Javac编译器的数据流分析工作保证了在方法执行的任何时候，操作数栈的深度都不会超过在max_stacks数据项中设定的最大值**。

**当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈和入栈操作**。譬如在做算术运算的时候是通过将运算涉及的操作数栈压入栈顶后调用运算指令来进行的，又譬如在调用其他方法的时候是通过操作数栈来进行方法参数的传递。举个例子，例如整数加法的字节码指令iadd，这条指令在运行的时候要求操作数栈中最接近栈顶的两个元素已经存入了两个int型的数值，当执行这个指令时，会把这两个int值出栈并相加，然后将相加的结果重新入栈。

操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，在编译程序代码的时候，编译器必须要严格保证这一点，在类校验阶段的数据流分析中还要再次验证这一点。

两个不同栈帧作为不同方法的虚拟机栈的元素，是完全相互独立的。但是在大多虚拟机的实现里都会进行一些优化处理，令两个栈帧出现一部分重叠。让下面栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起，这样做不仅节约了一些空间，更重要的是在进行方法调用时就可以直接共用一部分数据，无须进行额外的参数复制传递了：

![Untitled](https://img.masaiqi.com/202209081944319.png)

## 1.3. 动态连接

**每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接（Dynamic Linking）。Class文件的常量池中存有大量的符号引用，字节码中的方法调用指令就以常量池里指向方法的符号引用作为参数。这些符号引用一部分会在类加载阶段或者第一次使用的时候就被转化为直接引用，这种转化被称为静态解析。另外一部分将在每一次运行期间都转化为直接引用，这部分就称为动态连接。**

## 1.4. 方法返回地址

**当一个方法开始执行后，只有两种方式退出这个方法：**

- **正常调用完成（Normal Method Invocation Completion）**：执行引擎遇到任意一个方法返回的字节码指令，这时候可能会有返回值传递给上层的方法调用者（调用当前方法的方法称为调用者或者主调方法），方法是否有返回值以及返回值的类型将根据遇到何种方法返回指令来决定。
- **异常调用完成（Abrupt Method Invocation Completion）**：在方法执行的过程中遇到了异常，并且这个异常没有在方法体内得到妥善处理。无论是Java虚拟机内部产生的异常，还是代码中使用athrow字节码指令产生的异常，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出。一个方法使用异常完成出口的方式退出，是不会给它的上层调用者提供任何返回值的。

**无论采用何种退出方式，在方法退出之后，都必须返回到最初方法被调用时的位置，程序才能继续执行：**

- **正常调用完成时可能需要在栈帧中保存一些信息，用来帮助恢复它的上层主调方法的执行状态**。一般来说，方法正常退出时，主调方法的PC计数器的值就可以作为返回地址，栈帧中很可能会保存这个计数器值。
- **异常调用完成时，返回地址是要通过异常处理器表来确定的，栈帧中就一般不会保存这部分信息**。

方法退出的过程实际上等同于把当前栈帧出栈，因此退出时可能执行的操作有：

- 恢复上层方法的局部变量表和操作数栈
- 把返回值（如果有的话）压入调用者栈帧的操作数栈中
- 调整PC计数器的值以指向方法调用指令后面的一条指令等。

## 1.5. 附加信息

**《Java虚拟机规范》允许虚拟机实现增加一些规范里没有描述的信息到栈帧之中，例如与调试、性能收集相关的信息，这部分信息完全取决于具体的虚拟机实现。**

# 2. 方法调用

方法调用并不等同于方法中的代码被执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法），暂时还未涉及方法内部的具体运行过程。

在程序运行时，进行方法调用是最普遍、最频繁的操作之一。

Class文件的编译过程中不包含传统程序语言编译的连接步骤，一切方法调用在Class文件里面存储的都只是符号引用，而不是方法在实际运行时内存布局中的入口地址（直接引用）。这个特性给Java带来了更强大的动态扩展能力，但也使得Java方法调用过程变得相对复杂，某些调用需要在类加载期间，甚至到运行期间才能确定目标方法的直接引用。

## 2.1. 解析

所有方法调用的目标方法在Class文件里面都是一个常量池中的符号引用，**“编译期可知，运行期不可变”的类，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这类方法的调用被称为解析（Resolution）。**

**解析调用一定是个静态的过程，在编译期间就完全确定，在类加载的解析阶段就会把涉及的符号引用全部转变为明确的直接引用，不必延迟到运行期再去完成。**

字节码指令集设计了不同的指令：

- invokestatic。用于调用静态方法。
- invokespecial。用于调用实例构造器<init>()方法、私有方法和父类中的方法。
- invokevirtual。用于调用所有的虚方法。
- invokeinterface。用于调用接口方法，会在运行时再确定一个实现该接口的对象。
- invokedynamic。先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。

**前4条调用指令，分派逻辑都固化在Java虚拟机内部，而invokedynamic指令的分派逻辑是由用户设定的引导方法来决定的。**

**区分非虚方法与虚方法：**

- **非虚方法”（Non-Virtual Method）：能被invokestatic和invokespecial指令调用的方法（都可以在解析阶段中确定唯一的调用版本，类加载的时候就可以把符号引用解析为该方法的直接引用）：静态方法、私有方法、实例构造器、父类方法4种，被final修饰的方法（历史原因，由invokevirtual指令调用）。**
- **虚方法（Virtual Method）：无法在解析阶段将符号引用转化为直接引用**

## 2.2. 分派

### 2.2.1. 静态分派

**所有依赖静态类型来决定方法执行版本的分派动作，都称为静态分派。静态分派的最典型应用表现就是方法重载**。静态分派发生在编译阶段，因此确定静态分派的动作实际上不是由虚拟机来执行的，这点也是为何一些资料选择把它归入“解析”而不是“分派”的原因。

### 2.2.2. 动态分派

动态分派的最典型代表是Java语言重写（Override）。

在虚拟机执行重写方法时，执行invokevirtual指令，不仅仅是将常量池中方法的符号引用解析到直接引用上，还需要根据方法的接收者的实际类型来确定方法的版本，这就是Java语言中方法重写的本质。

### 2.2.3. 单分派与多分排

方法的接收者与方法的参数统称为方法的宗量：

- 单分派是根据一个宗量对目标方法进行选择
- 多分派则是根据多于一个宗量对目标方法进行选择。

```java
public class Dispatch {

    static class QQ {
    }
    
    static class _360 {
    }
    
    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("fatherchooseqq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("fatherchoose360");
        }
    }
    
    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("sonchooseqq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("sonchoose360");
        }
    }

    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();
        father.hardChoice(new _360());
        son.hardChoice(new QQ());
    }
}
```

结果：

```java
father choose 360
son choose qq
```

在main()方法里调用了两次hardChoice方法：

- **静态分派过程**：

  两个宗量：

  1. 静态类型（声明变量时的类型）是Father还是Son
  2. 方法参数是QQ还是360

  **最终产生两条invokevirtual指令，分别指向常量池Father::hardChoice(360)及Father::hardChoice(QQ)方法的符号引用**

- **动态分派过程：**

  **在上文基础上，只关注执行invokevirtual指令时，该方法的实际接受者是Father还是Son。**

**总结：Java语言是一门静态多分派、动态单分派的语言。**

# 3. 动态类型语言支持

## 3.1. 什么叫做“动态类型语言”

**动态类型语言的关键特征是它的类型检查的主体过程是在运行期而不是编译期进行的。**

比如：

```java
Object a = new Object();
a.doSomething(); // error
```

这里a必须拥有对应方法才不会报错，因为Java中会编译时进行检查。同样的写法在JS中则不会报错。

**JDK 7以前的字节码指令集中，4条方法调用指令（invokevirtual、invokespecial、invokestatic、invokeinterface）的第一个参数都是被调用的方法的符号引用，方法的符号引用在编译时产生，而动态类型语言只有在运行期才能确定方法的接收者。**

## 3.2. java.lang.invoke包

JDK7时加入了java.lang.invoke包（JSR-292一部分），这个包的主要目的是在之前单纯依靠符号引用来确定调用的目标方法这条路之外，提供一种新的动态确定目标方法的机制，称为“方法句柄”（Method Handle）：

```java
/**
 * @author <a href="mailto:masaiqi.com@gmail.com">masaiqi</a>
 * @date 2021/4/20 11:29
 */
public class MethodHandleTest {

    static class ClassA {
        public void println(String s) {
            System.out.println(s);
        }
    }

    public static void main(String[] args) throws Throwable {
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
        getPrintlnMH(obj).invokeExact("hello");
    }

    private static MethodHandle getPrintlnMH(Object receiver) throws Throwable {
        // MethodType：代表“方法类型”，包含了方法的返回值（methodType()的第一个参数）和 具体参数（methodType()第二个及以后的参数）
        MethodType mt = MethodType.methodType(void.class, String.class);
        // lookup()方法来自于MethodHandles.lookup，这句的作用是在指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄。
        // 因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，也即this指向的对象，这个参数以前是放在参数列表中进行传递，现在提供了bindTo()方法来完成这件事情。
        return lookup().findVirtual(receiver.getClass(), "println", mt).bindTo(receiver);
    }

}
```

主要这里的“虚方法”上文有提及，即无法在解析阶段直接将符号引用转化为直接引用。

**MethodHandle与Reflection（反射）的区别：**

- **Reflection和MethodHandle机制本质上都是在模拟方法调用，但是Reflection是在模拟Java代码层次的方法调用，而MethodHandle是在模拟字节码层次的方法调用**。在MethodHandles.Lookup上的3个方法findStatic()、findVirtual()、findSpecial()正是为了对应于invokestatic、invokevirtual（以及invokeinterface）和invokespecial这几条字节码指令的执行权限校验行为，而这些底层细节在使用Reflection API时是不需要关心的。
- **Reflection是重量级，而MethodHandle是轻量级**。Reflection中的java.lang.reflect.Method对象远比MethodHandle机制中的java.lang.invoke.MethodHandle对象所包含的信息来得多。**前者是方法在Java端的全面映像，包含了方法的签名、描述符以及方法属性表中各种属性的Java端表示方式，还包含执行权限等的运行期信息。而后者仅包含执行该方法的相关信息**。
- **由于MethodHandle是对字节码的方法指令调用的模拟，那理论上虚拟机在这方面做的各种优化（如方法内联），在MethodHandle上也应当可以采用类似思路去支持**（但目前实现还在继续完善中），而**通过反射去调用方法则几乎不可能直接去实施各类调用点优化措施**。
- **Reflection API的设计目标是只为Java语言服务的，而MethodHandle则设计为可服务于所有Java虚拟机之上的语言，其中也包括了Java语言而已**。

## 3.3. invokeDynamic指令

**某种意义上可以说invokedynamic指令与MethodHandle机制的作用是一样的，只是MethodHandle用上层代码和API来实现，invokedynamic则用字节码和Class中其他属性、常量来完成**。

# 4. 基于栈的字节码解释执行引擎

整体执行过程与传统编译原理中类似（包含解释执行&生成目标机器代码）：

![Untitled](https://img.masaiqi.com/202209081944355.png)

## 4.1. 基于栈的指令集与基于寄存器的指令集

**Javac编译器输出的字节码指令流，基本上是一种基于栈的指令集架构（Instruction Set Architecture，ISA），字节码指令流里面的指令大部分都是零地址指令，它们依赖操作数栈进行工作。**

计算1+1:

- 在基于栈的指令集中：

  ```bash
  iconst_1
  iconst_1
  iadd
  istore_0
  ```

  两条iconst_1指令压入操作数1和1，iadd指令让栈顶两个值出栈，相加送回栈顶，istore_0指令把栈顶的值放到局部变量表的第0个变量槽中。（iadd指令无需操作数，依赖栈完成数据存储）

- 基于寄存器的指令集：

  ```bash
  mov eax, 1
  add eax, 1
  ```

  mov指令把EAX寄存器的值设为1，然后add指令再把这个值加1，结果就保存在EAX寄存器中。（mov指令 + 两个操作数模式，依赖寄存器访问/存储数据）

**基于栈的指令集&基于寄存器的指令集的优缺点：**

- **基于栈的指令集易于移植。寄存器由硬件直接提供，受到硬件约束。**
- **基于栈的指令集执行速度偏慢。（只针对解释执行，如果经过即时编译器输出成物理机上的汇编指令流，就无区别了。）**
- **基于栈的指令集代码紧凑，但是完成一个功能所需指令数比寄存器架构多。**
- **栈实现在内存中，内存访问存在性能瓶颈；寄存器架构下，寄存器作为CPU中的高速存储部件，性能比内存高。**

## 4.2. 基于栈的解释器执行过程DEMO

**PS：下文只是一个概念模型，实际场景下虚拟机可能有优化。**

Java代码：

```java
public int calc() {
	int a = 100;
	int b = 200;
	int c = 300;
	return (a+b) * c;
}
```

javap后：

```bash
public int calc();
    descriptor: ()I
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=1
         0: bipush        100
         2: istore_1
         3: sipush        200
         6: istore_2
         7: sipush        300
        10: istore_3
        11: iload_1
        12: iload_2
        13: iadd
        14: iload_3
        15: imul
        16: ireturn
      LineNumberTable:
        line 10: 0
        line 11: 3
        line 12: 7
        line 13: 11
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      17     0  this   Lcom/masaiqi/StackProcessTest;
            3      14     1     a   I
            7      10     2     b   I
           11       6     3     c   I
```

- stack=2，即需要深度为2的操作数栈
- locals=4，即需要4个变量槽的局部变量空间

在执行偏移地址为0的指令情况下：

- 首先执行偏移地址为0的指令，Bipush指令的作用是将单字节的整型常量值推入操作数栈顶，带一个操作数100，说明需要将100这个常量值推入栈顶。

  ![Untitled](https://img.masaiqi.com/202209081944371.png)

- 执行偏移地址为2的指令，istore_1指令的作用是将操作数栈顶的整型值出栈并存放到第一个局部变量槽中（截止到偏移量10为止都是做这件事）。也就是代码中的a、b、c赋值100、200、300。

  ![Untitled](https://img.masaiqi.com/202209081944386.png)

- 执行偏移地址为11的指令，iload_1指令的作用是将局部变量表第1个变量槽中的整型值复制道操作数栈顶。

  ![Untitled](https://img.masaiqi.com/202209081944399.png)

- 执行偏移地址为12的指令，iload_2指令的作用是将局部变量表第2个变量槽中的整型值复制道操作数栈顶。

  ![Untitled](https://img.masaiqi.com/202209081944416.png)

- 执行偏移地址为13的指令，iadd指令将操作数栈中头两个栈顶元素出栈，做加法，将结果300重新入栈。

  ![Untitled](https://img.masaiqi.com/202209081944435.png)

- 执行偏移地址为14的指令，iload_3指令的作用是将局部变量表第3个变量槽中的整型值复制道操作数栈顶。

  ![Untitled](https://img.masaiqi.com/202209081944461.png)

- 执行偏移地址为15的指令，imul指令将操作数栈中头两个栈顶元素出栈，做乘法法，将结果90000重新入栈。

- 执行偏移地址为16的指令，ireturn指令是方法返回指令之一，它将结束方法并将操作数栈顶整型值返回给方法的调用者。

  ![Untitled](https://img.masaiqi.com/202209081944484.png)