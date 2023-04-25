---
title: "volatile学习笔记"
date: 2020-06-16 21:57:18.0
draft: false
type: "post"
showTableOfContents: true
tags: ["Java","虚拟机"]
---

# 1. volatile关键字的作用

- **可见性**：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
- **有序性（阻止指令重排）**：volatile标志的变量的写不能与之前的代码重排序；volatile标志的变量的读不能与之后的代码进行重排序(一般记为 **写前读后** 后文详细说明)
- **原子性**：对**任意单个volatile变量的读/写具有原子性**，但类似于volatile++这种复合操作不具有原子性。

# 2. volatile关键字的原理

我们声明一个volatile类型的变量：

```java
volatile instance=newSingleton()
```

查看对应的汇编代码如下：

```
0x01a3de1d: movb $0×0,0×1104800(%esi);0x01a3de24: lock addl $0×0,(%esp);
```

可以看到中间有一个**lock**指令

关于lock指令，描述如下：

- **确保对内存的读改写操作原子执行。**
- **禁止该指令，与之前和之后的读和写指令重排序。**
- **把写缓冲区中的所有数据刷新到内存中，并使其他处理器中的缓存无效。**

如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

**如果一个字段被声明成volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的。**

# 3. volatile关键字的内存语义

volatile写的内存语义如下:

**当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。**

volatile读的内存语义如下:

**当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。**

# 4. volatile关键字的内存语义实现

JMM存在重排序(JMM相关，这里不讨论)，但是**为了实现volatile的内存语义，JMM会对重排序进行限制**：

![volatile%20fae817c5399d43ce9d42623083f90b86/Untitled.png](https://img.masaiqi.com/20200616215311.png)

从上述表格可以得出以下总结（**写前读后**）：

- 当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。
- 当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。

**为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序**。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能。为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略：

- 在每个volatile写操作的前面插入一个StoreStore屏障。
- 在每个volatile写操作的后面插入一个StoreLoad屏障。
- 在每个volatile读操作的后面插入一个LoadLoad屏障。
- 在每个volatile读操作的后面插入一个LoadStore屏障。

上述内存屏障插入策略非常保守，但它可以保证在任意处理器平台，任意的程序中都能得到正确的volatile内存语义。

这些内存屏障指令作用如图：

![volatile%20fae817c5399d43ce9d42623083f90b86/Untitled%203.png](https://img.masaiqi.com/20200616215314.png)

![volatile%20fae817c5399d43ce9d42623083f90b86/Untitled%204.png](https://img.masaiqi.com/20200616215315.png)

# 5. volatile关键字与happens-before规则

happens-before规则中有一条：

**volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。**

这一块有很多可以讨论的，这篇文章写volatile，我们就讨论volatile规则的语义，可以简单理解为**volatile的写一定对后续volatile的读可见**。

# 6. volatile原子性的讨论

volatile讨论原子性的前提一定要注意，这里讨论的是：

**任意单个volatile变量的读/写具有原子性**

比如`i++`这种操作，volatile尽管保证了单次修改的原子性，但是i++包含3步：

1. 读取缓存中的数据
2. 在缓存中修改值+1
3. 刷新到主存中

这3个操作不是一个完整的原子操作，在进入第2步之前，是还不存在总线锁/缓存锁的，因为没有发生修改，因此会出现下面的情况(画图比较麻烦。。干脆用ipad手画了一张。。应该可以看懂)：

![volatile%20fae817c5399d43ce9d42623083f90b86/Untitled%205.png](https://img.masaiqi.com/20200616215316.png)

在上面的流程中，我们用圆圈表示顺序，假设是第1步和第2步这样的顺序，尽管volatile的语义保证了各自单次读取的原子性，但是这里读取到都是0；接下来就是分别获得锁去+1，最后i只被加了一次。

除此以外，**需要注意volatile即使标在long和double这种64位的变量上依旧是可以具有原子性**的(我们知道，一般的64位的类型在32位系统中会被拆分为两个32位因此不具备普遍的原子性)

这一点在Oracle的[Java Language Specification](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.7)中有提到：

![volatile%20fae817c5399d43ce9d42623083f90b86/Untitled%201.png](https://img.masaiqi.com/20200616215312.png)

《Java并发编程的艺术》一书的作者也有提到：

![volatile%20fae817c5399d43ce9d42623083f90b86/Untitled%202.png](https://img.masaiqi.com/20200616215313.png)

# 7. 参考

- [读懂Java中的volatile-五月的仓颉博客](https://www.cnblogs.com/xrq730/p/7048693.html)
- [CPU缓存一致性协议](https://blog.csdn.net/m18870420619/article/details/82431319)
- 《Java并发编程的艺术》