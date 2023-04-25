---
title: "《深入理解Java虚拟机》第三版 - 01 - Java内存区域与内存溢出异常"
date: 2022-09-08 19:37:40.467
draft: false
type: "post"
showTableOfContents: true
tags: ["Java","虚拟机"]
---

# 1. JVM运行时数据区

根据《Java虚拟机规范》：

![Untitled](https://img.masaiqi.com/202209081935871.png)

## 1.1. 程序计数器

**程序计数器（Program Counter Register）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器**。在Java虚拟机的概念模型里，字节码解释器工作时就是**通过改变这个计数器的值来选取下一条需要执行的字节码指令，它是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成**。

在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。因此，**为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。**

**此内存区域是唯一一个在《Java虚拟机规范》中没有规定任何OutOfMemoryError情况的区域。**

## 1.2. Java虚拟机栈

**与程序计数器一样，Java虚拟机栈（Java Virtual Machine Stack）也是线程私有的，它的生命周期与线程相同**。

**虚拟机栈描述的是Java方法执行的线程内存模型：每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧用于存储局部变量表、操作数栈、动态连接、方法出口等信息。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程**。

通常说的“栈”是指虚拟机栈，或者虚拟机栈中局部变量表部分。

**局部变量表存放了编译期可知的各种Java虚拟机基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它并不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）**。

这些数据类型在局部变量表中的存储空间以局部变量槽（Slot）来表示，其中64位长度的long和double类型的数据会占用两个变量槽，其余的数据类型只占用一个。

这里有一个注意点：

<aside>
💡 对于64位的数据类型，虚拟机会以高位在前的方式为其分配两个连续的Slot空间，这一做法与"long和double的非原子性协定" 中把一次long 和double 数据类型读写分割为两次32位读写的做法类似。**不过，由于局部变量表建在线程的堆栈上，是线程私有的数据，无论读写两个连续的Slot是否是原子操作，都不会引起数据安全问题**。

</aside>

**局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多少变量槽来存储局部变量表是完全确定的，在方法运行期间不会改变局部变量表的变量槽数量**。**但是每个变量槽的实际内存空间大小由具体虚拟机实现决定**。

在《Java虚拟机规范》中，对这个内存区域规定了两类异常状况：

- **如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常**；
- **如果Java虚拟机栈容量可以动态扩展（HotSpot不会），当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError异常**。

## 1.3. 本地方法栈

本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别：

- **虚拟机栈为虚拟机执行Java方法（也就是字节码）服务**
- **本地方法栈则是为虚拟机使用到的本地（Native）方法服务**

《Java虚拟机规范》对本地方法栈中方法使用的语言、使用方式与数据结构并没有任何强制规定，因此具体的虚拟机可以根据需要自由实现它，甚至有的Java虚拟机（譬如HotSpot虚拟机）直接就把本地方法栈和虚拟机栈合二为一。

**与虚拟机栈一样，本地方法栈也会在栈深度溢出或者栈扩展失败时分别抛出StackOverflowError和OutOfMemoryError异常**。

## 1.4. Java堆

**Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，Java世界里“几乎”所有的对象实例都在这里分配内存**。

Java堆是垃圾收集器管理的内存区域，因此一些资料中它也被称作“GC堆”（GarbageCollectedHeap）。

由于现代垃圾收集器大多基于分代收集理论，堆上会有一些诸如老年代，新生代之类的名词，但是现在HotSpot虚拟机里面也出现了不采用分代设计的新垃圾收集器，因此这样描述不够准确。

**Java堆既可以被实现成固定大小的，也可以是可扩展的，不过当前主流的Java虚拟机都是按照可扩展来实现的（通过参数Xmx和Xms设定）。如果在Java堆中没有内存完成实例分配，并且堆也无法再扩展时，Java虚拟机将会抛出OutOfMemoryError异常**。

## 1.5. 方法区

**方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据**。

虽然《Java虚拟机规范》中把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫作“非堆”（NonHeap），目的是与Java堆区分开来。

**JDK8以前，HotSpot虚拟机用永久代来实现方法区，因此会把方法区会被称为永久代，这种设计导致了Java应用更容易遇到内存溢出的问题（永久代有-XX：MaxPermSize的上限，即使不设置也有默认大小），会抛出OutOfMemoryError: *PermGen* space异常**。

**到了JDK8，终于完全废弃了永久代的概念，改用在本地内存中实现的元空间（Metaspace）来代替。**

元空间存在于本地内存，意味着只要本地内存足够，就不会出现异常错误，JVM也提供了类似-XX:MetaspaceXX等参数控制元空间大小，默认情况下是无限的，取决于系统的实际可用空间。

**根据《Java虚拟机规范》的规定，如果方法区无法满足新的内存分配需求时，将抛出OutOfMemoryError异常。**

## 1.6. 运行时常量池

**运行时常量池（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table），用于存放编译期生成的各种字面量与符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中**。

对于运行时常量池，《Java虚拟机规范》并没有做任何细节的要求，不同提供商实现的虚拟机可以按照自己的需要来实现这个内存区域。

**运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是说，并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可以将新的常量放入池中，这种特性被开发人员利用得比较多的便是String类的intern()方法**

<aside>
💡 关于java.lang.String#intern方法，官方文档中有说：
When the intern method is invoked, if the pool already contains a string equal to this `String` object as determined by the `[equals(Object)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/String.html#equals(java.lang.Object))` method, then the string from the pool is returned. Otherwise, this `String` object is added to the pool and a reference to this `String` object is returned.

</aside>

**由于运行时常量池是方法区的一部分，也受方法区的规定，无法满足新的内存分配需求时，将抛出OutOfMemoryError异常。**

## 1.7. 直接内存

**直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是《Java虚拟机规范》中定义的内存区域。但是这部分也可能导致OutOfMemoryError异常出现**。

在JDK1.4中新加入了NIO（NewInput/Output）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它**可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据**。显然，本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，则肯定还是**会受到本机总内存（包括物理内存、SWAP分区或者分页文件）大小以及处理器寻址空间的限制**，**一般服务器管理员配置虚拟机参数时，会根据实际内存去设置Xmx等参数信息，但经常忽略掉直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现OutOfMemoryError异常。**

# 2. HotSpot虚拟机

## 2.1. 对象的创建

1. Java虚拟机执行new指令
2. 类加载检查，如果没有加载，则需要先执行类加载过程
3. 分配内存对象，对象所需内存大小在类加载完成后可确定

可能在分配内存时的线程安全问题：

1. CAS + 失败重试 保证原子性
2. **预选在堆中分配一小块内存作为缓冲（ThreadLocal Allocation Buffer，TLAB），线程先在本地缓冲区分配，分配新的缓存区时才需要同步锁定。虚拟机是否使用TLAB，可以通过XX：+/UseTLAB参数来设定。**

## 2.2. 对象的内存布局

**在HotSpot虚拟机里，对象在堆内存中的存储布局可以划分为三个部分**

- **对象头（Header）：MarkWord与类型指针**
- **实例数据（Instance Data）：对象真正存储的有效信息**
- **对齐填充（Padding）：为了位数对齐而进行填充，无特别意义。**

对象头分为两部分：

- **MarkWord，一个有着动态定义的数据结构，以便在极小的空间内存储尽量多的数据，根据对象的状态复用自己的存储空间**。例如在32位的HotSpot虚拟机中，如对象未被同步锁锁定的状态下，MarkWord的32个比特存储空间中的25个比特用于存储对象哈希码，4个比特用于存储对象分代年龄，2个比特用于存储锁标志位，1个比特固定为0，在其他状态（轻量级锁定、重量级锁定、GC标记、可偏向）下对象的存储内容：
  
    ![Untitled](https://img.masaiqi.com/202209081935897.png)
    
- **类型指针，即对象指向它的类型元数据的指针，Java虚拟机通过这个指针来确定该对象是哪个类的实例。**并不是所有的虚拟机实现都必须在对象数据上保留类型指针，换句话说，查找对象的元数据信息并不一定要经过对象本身。此外，如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是如果数组的长度是不确定的，将无法通过元数据中的信息推断出数组的大小。

## 2.3. 对象的访问定位

创建对象自然是为了后续使用该对象，我们的Java程序会通过栈上的reference数据来操作堆上的具体对象。由于reference类型在《Java虚拟机规范》里面只规定了它是一个指向对象的引用，并没有定义这个引用应该通过什么方式去定位、访问到堆中对象的具体位置，所以**对象访问方式也是由虚拟机实现而定的，主流的访问方式主要有使用句柄和直接指针两种：**

- 句柄访问
  
    **Java堆中将可能会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自具体的地址信息。**
    
    **使用句柄来访问的最大好处就是reference中存储的是稳定句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要被修改。**
    
    ![Untitled](https://img.masaiqi.com/202209081935916.png)
    
- 直接指针
  
    **Java堆中对象的内存布局就必须考虑如何放置访问类型数据的相关信息，reference中存储的直接就是对象地址，如果只是访问对象本身的话，就不需要多一次间接访问的开销**。
    
    **使用直接指针来访问最大的好处就是速度更快，它节省了一次指针定位的时间开销，由于对象访问在Java中非常频繁，因此这类开销积少成多也是一项极为可观的执行成本**（HotSpot主要使用的方式）
    
    ![Untitled](https://img.masaiqi.com/202209081935946.png)
    

# 3. OutOfMemoryError异常

这一块与原书中会有一些差异，笔者使Oracle的JDK11进行测试：

![Untitled](https://img.masaiqi.com/202209081935959.png)

此外需要注意启动的VM参数会在注释中。

## 3.1. Java堆溢出

Java堆用于储存对象实例，只要**不断地创建对象，并且保证GCRoots到对象之间有可达路径来避免垃圾回收机制清除这些对象，那么随着对象数量的增加，总容量触及最大堆的容量限制后就会产生内存溢出异常。**

说明一下几个参数：

- `-XX:+HeapDumpOnOutOfMemoryError`：让虚拟机在出现内存溢出时Dump出当前内存堆转储快找以便进行事后分析。
- `-Xms`最小堆的大小
- `-Xmx`最大堆的大小43

```java
package com.masaiqi.oom;

import java.util.ArrayList;
import java.util.List;

/**
 * 模拟堆OOM异常
 * <p>
 * VMArgs：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 *
 * @author <a href="mailto:masaiqi.com@gmail.com">masaiqi</a>
 * @date 2021/12/20 20:25
 */
public class HeapError {
    public static void main(String[] args) {
        List<Object> test = new ArrayList<>();
        for (; ; ) {
            test.add(new Object());
        }
    }
}
```

![Untitled](https://img.masaiqi.com/202209081935974.png)

**这个导出的`**.hprof`文件可以用Eclipse Memory Analyzer Tool（MAT）或VisualVm进行分析**，笔者这里使用VisualVm进行分析：

![Untitled](https://img.masaiqi.com/202209081935992.png)

- **如果是内存泄漏，则需要查看泄漏对象到GC Roots到引用链，找到泄漏对象是什么样的路径导致垃圾收集器无法回收他们**。
- **如果不是内存泄漏，也就是内存中的对象确实都是必须存活的，就应该调整一下虚拟机参数**。

## 3.2. 虚拟机栈和本地方法栈溢出

由于**HotSpot虚拟机中并不区分虚拟机栈和本地方法栈**，因此**对于HotSpot来说，**`-Xoss`**参数（设置本地方法栈大小）虽然存在，但实际上是没有任何效果的，栈容量只能由**`-Xss`**参数来设定**。

关于虚拟机栈和本地方法栈，在《Java虚拟机规范》中描述了两种异常： 

- **如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常**。
- **如果虚拟机的栈内存允许动态扩展（允许虚拟机自行选择是否支持栈动态扩展），当扩展栈容量无法申请到足够的内存时，将抛出OutOfMemoryError异常**。其中，***HotSpot虚拟机不支持栈内存的动态拓展，因此除非在创建线程申请内存时就因无法获得足够内存而出现OutOfMemoryError异常，否则在线程运行时是不会因为扩展而导致内存溢出的，只会因为栈容量无法容纳新的栈帧而导致StackOverflowError异常***。
- 验证栈深度大于虚拟机所允许最大深度时异常：
  
    ```java
    package com.masaiqi.oom;
    
    /**
     * 模拟栈异常 - 栈深度大于虚拟机所允许的最大深度
     *<p>
    * VMArgs：-Xss144k
     *
     *@author<a href="mailto:masaiqi.com@gmail.com">masaiqi</a>
    *@date2021/12/20 22:56
     */
    public class StackErrorA {
    
        private int stackLength = 1;
    
        public void stackLeak() {
            stackLength++;
            stackLeak();
        }
    
        public static void main(String[] args) {
            StackErrorA stackErrorA = new StackErrorA();
            try {
                stackErrorA.stackLeak();
            }catch (Throwable e) {
                System.out.println("Stack length: " + stackErrorA.stackLength);
                throw e;
            }
        }
    }
    
    ```
    
    ![Untitled](https://img.masaiqi.com/202209081935005.png)
    
- 验证创建线程时，通过占用大量局部变量表，进而导致申请虚拟机栈内存时异常
  
    ```java
    package com.masaiqi.oom;
    
    /**
     * 模拟栈异常 - 创建线程时，通过占用大量局部变量表，进而导致申请虚拟机栈内存时异常
     *<p>
    * VMArgs：-Xss144k
     *
     *@author<a href="mailto:masaiqi.com@gmail.com">masaiqi</a>
    *@date2021/12/20 22:56
     */
    public class StackErrorB {
    
        private static intstackLength= 0;
    
        public static void test() {
            long unused1, unused2, unused3, unused4, unused5,
                    unused6, unused7, unused8, unused9, unused10,
                    unused11, unused12, unused13, unused14, unused15,
                    unused16, unused17, unused18, unused19, unused20,
                    unused21, unused22, unused23, unused24, unused25,
                    unused26, unused27, unused28, unused29, unused30,
                    unused31, unused32, unused33, unused34, unused35,
                    unused36, unused37, unused38, unused39, unused40,
                    unused41, unused42, unused43, unused44, unused45,
                    unused46, unused47, unused48, unused49, unused50,
                    unused51, unused52, unused53, unused54, unused55,
                    unused56, unused57, unused58, unused59, unused60,
                    unused61, unused62, unused63, unused64, unused65,
                    unused66, unused67, unused68, unused69, unused70,
                    unused71, unused72, unused73, unused74, unused75,
                    unused76, unused77, unused78, unused79, unused80,
                    unused81, unused82, unused83, unused84, unused85,
                    unused86, unused87, unused88, unused89, unused90,
                    unused91, unused92, unused93, unused94, unused95,
                    unused96, unused97, unused98, unused99, unused100;
    		stackLength++;
    		test();
            unused1 = unused2 = unused3 = unused4 = unused5 =
            unused6 = unused7 = unused8 = unused9 = unused10 =
            unused11 = unused12 = unused13 = unused14 = unused15 =
            unused16 = unused17 = unused18 = unused19 = unused20 =
            unused21 = unused22 = unused23 = unused24 = unused25 =
            unused26 = unused27 = unused28 = unused29 = unused30 =
            unused31 = unused32 = unused33 = unused34 = unused35 =
            unused36 = unused37 = unused38 = unused39 = unused40 =
            unused41 = unused42 = unused43 = unused44 = unused45 =
            unused46 = unused47 = unused48 = unused49 = unused50 =
            unused51 = unused52 = unused53 = unused54 = unused55 =
            unused56 = unused57 = unused58 = unused59 = unused60 =
            unused61 = unused62 = unused63 = unused64 = unused65 =
            unused66 = unused67 = unused68 = unused69 = unused70 =
            unused71 = unused72 = unused73 = unused74 = unused75 =
            unused76 = unused77 = unused78 = unused79 = unused80 =
            unused81 = unused82 = unused83 = unused84 = unused85 =
            unused86 = unused87 = unused88 = unused89 = unused90 =
            unused91 = unused92 = unused93 = unused94 = unused95 =
            unused96 = unused97 = unused98 = unused99 = unused100 = 0;
        }
    
        public static void main(String[] args) {
            try {
    						test();
            } catch (Error e) {
                System.out.println("Stack length: " +stackLength);
                throw e;
            }
        }
    }
    
    ```
    
    ![Untitled](https://img.masaiqi.com/202209081935025.png)
    

结果表明，**HotSpot虚拟机中，无论是由于栈帧太大还是虚拟机栈容量太小，当新的栈帧内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常**。

## 3.3. 方法区和运行时常量池溢出

### 3.3.1 运行时常量池溢出

使用String#intern进行测试。

**在JDK6或更早的HotSpot虚拟机中，常量池分配都是在永久代中**，因此可以通过`-XX：PermSize`和`-XX：MaxPermSize`限制永久代的大小，即可间接限制其中常量池的容量，在溢出时，会有如下异常：

```java
Exceptioninthread"main"java.lang.OutOfMemoryError: PermGenspace
```

**HotSpot从JDK7开始逐步“去永久代”的计划，在JDK8中完全使用元空间代替永久代，因此在JDK7中使用`-XX：MaxPermSize`不会重现错误；在JDK8及以上版本使用用`-XX：MaxMetaspaceSize`限制元空间大小也不会重现错误。**

**因为自JDK7开始，原本存在永久代的字符串常量池被移到Java堆之中，需要使用-Xmx参数限制最大堆容量来重现：**

```java
Exceptioninthread"main"java.lang.OutOfMemoryError:Javaheapspace
```

### 3.3.2. 其他方法区相关内容

方法区的主要职责是用于存放类型的相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。这部分采用Cglib字节码提升技术进行演示，这在Spring framework等框架中被广泛使用：

```java
package com.masaiqi.oom;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * 模拟方法区异常
 * <p>
 * JDK7 VMArgs：-XX:PermSize=10M -XX:MaxPermSize=10M
 * <p>
 * JDK8+ VMArgs：-XX:MaxMetaspaceSize=10M
 *
 *
 * @author <a href="mailto:masaiqi.com@gmail.com">masaiqi</a>
 * @date 2021/12/28 20:14
 */
public class MethodAreaOOM {

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }

    static class OOMObject {
    }

}
```

在JDK7中会有如下报错：

```java
Caused by:java.lang.OutOfMemoryError:PermGenspace
```

在JDK8中会有如下报错：

![Untitled](https://img.masaiqi.com/202209081935040.png)

在JDK8以后，永久代便完全退出了历史舞台，元空间作为其替代者登场，HotSpot还是提供了一些参数作为元空间的防御措施，主要包括：

- `-XX：MaxMetaspaceSize`：设置元空间最大值，默认是1，即不限制，或者说只受限于本地内存大小。
- `-XX：MetaspaceSize`：指定元空间的初始空间大小，以字节为单位，达到该值就会触发垃圾收集进行类型卸载，同时收集器会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过XX：MaxMetaspaceSize（如果设置了的话）的情况下，适当提高该值。
- `-XX：MinMetaspaceFreeRatio`：作用是在垃圾收集之后控制最小的元空间剩余容量的百分比，可减少因为元空间不足导致的垃圾收集的频率。类似的还有`-XX：MaxMetaspaceFreeRatio`，用于控制最大的元空间剩余容量的百分比。

## 3.4. 本地直接内存溢出

直接内存（DirectMemory）的容量大小可通过`-XX：MaxDirectMemorySize`参数来指定，如果不去指定，则默认与Java堆最大值（由-Xmx指定）一致。

# 参考资料

- [JVM之局部变量表](https://blog.csdn.net/wuzhiwei549/article/details/80636404)
- [JVM为什么使用元空间替换了永久代](https://zhuanlan.zhihu.com/p/111809384)