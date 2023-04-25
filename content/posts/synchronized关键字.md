---
title: "synchronized关键字"
date: 2020-06-18 22:41:06.0
draft: false
type: "post"
showTableOfContents: true
tags: ["Java","虚拟机"]
---

# 1. 首先挂一个图

图片摘自[美团技术团队](https://tech.meituan.com/2018/11/15/java-lock.html)，个人觉得写的特别好：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled.png](https://img.masaiqi.com/20200618223900.png)

# 2. synchronized的锁作用范围

Java锁作用范围有两种：

- 一种是类的实例对象的锁(对象锁)。
- 一种是类的class对象(类锁)。

比如：

- 在一个静态方法前面加synchronized关键字，就是对这个类的class对象加锁。

```java
public class Do {
	public synchronized void test() {
	}
}
```

- 在代码块中，synchronized关键字中写的内容是对象实例的时候，就是对这个对象实例加锁。

```java
public class Do {
	
	Object object = new Object();

	public void test() {
		synchronized (object) {
			System.err.println(1);
		}
	}
}
```

# 3. synchronized的底层实现

从JVM规范中可以看到Synchonized在JVM里的实现原理，JVM**基于进入和退出Monitor对象来实现方法同步和代码块同步**，但两者的实现细节不一样。代码块同步是使用monitorenter和monitorexit指令实现的，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明。但是，方法的同步同样可以使用这两个指令来实现。

**monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处，JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有一个monitor与之关联，当且一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。**

**synchronized用的锁是存在Java对象头里的**。

如果对象是数组类型，则虚拟机用3个字宽（Word）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，1字宽等于4字节，**Java对象头存储结构如图所示：**

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%201.png](https://img.masaiqi.com/20200618223908.png)

Java对象头里的Mark Word里默认存储对象的HashCode、分代年龄和锁标记位。**32位JVM的MarkWord的默认存储结构如图所示**：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%202.png](https://img.masaiqi.com/20200618223912.png)

运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化，最终我们会得到下图：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%203.png](https://img.masaiqi.com/20200618223915.png)

在64位JVM中，有下图：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%204.png](https://img.masaiqi.com/20200618223919.png)

可以看到，无论是64位还是32位JVM，锁的标志位的定义都是相同的。

我们先引入一个包查看对象头的布局：

```xml
<dependency> 
	<groupId>org.openjdk.jol</groupId> 
	<artifactId>jol-core</artifactId> 
	<version>0.10</version>
</dependency>
```

我们用一个简单例子说明一下怎么使用：

```java
public class Test {
	public static void main(String[] args) {
		Lock lock = new Lock();
		synchronized (lock) {
			System.out.println(ClassLayout.parseInstance(lock).toPrintable());
		}
	}
}
```

控制台输出：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%205.png](https://img.masaiqi.com/20200618225234.png)

这里采用的是 `小端存储`(低位字节放在内存低地址中)，因此，我们有如下顺序

```java
00000001 10101000 10101001 10010000
```

可以看到，这里最后3位是001，也就是轻量锁。

这里采用的是lock实例的对象锁，锁状态存储在对应实例的对象头中。

# 4. 锁的膨胀（升级）

在Java中，**锁会随着竞争状态进行升级，但是锁只可以升级不能降级**：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%206.png](https://img.masaiqi.com/20200618223923.png)

## 4.1. 偏向锁

在大多数情况下，锁不仅仅不存在多线程的竞争，而且总是由同一个线程多次获得。在这个背景下就设计了偏向锁。偏向锁，顾名思义，就是锁偏向于某个线程。

**偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。**

当一个线程访问加了同步锁的代码块时，会在对象头中存储当前线程的ID，后续这个线程进入和退出这段加了同步锁的代码块时，不需要再次加锁和释放锁。而是直接比较对象头里面是否存储了指向当前线程的偏向锁。如果相等表示偏向锁是偏向于当前线程的，就不需要再尝试获得锁了，引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径。（偏向锁的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。)

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%207.png](https://img.masaiqi.com/20200618223927.png)

举个例子，下面代码，演示了一个偏向锁

偏向锁在jdk1.8的时候的时候是默认启动的，但是默认会在应用启动几秒后生效，我们这里通过参数取消延迟：`-XX:BiasedLockingStartupDelay=0`

```java
public class Test {
   public static void main(String[] args) {
      Lock lock = new Lock();
      synchronized (lock) {
         System.out.println(ClassLayout.parseInstance(lock).toPrintable());
      }
   }
}
```

控制台输出如下：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%208.png](https://img.masaiqi.com/20200618225311.png)

可以看到，这里最后三位是101 ，也就是偏向锁。

下面我们用两个线程去竞争锁资源，期望让他膨胀：

```java
public class Test {
   public static void main(String[] args) {
      Lock lock = new Lock();
      new Thread(() -> doJob(lock), "t1").start();
      new Thread(() -> doJob(lock), "t2").start();
   }

   public static void doJob(Lock lock) {
      synchronized (lock) {
         System.out.println(ClassLayout.parseInstance(lock).toPrintable());
      }
   }

}
```

控制台输出：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%209.png](https://img.masaiqi.com/20200618225226.png)

最后3位是010，也就是重量级锁，达到期望预期。

## 4.2. 轻量级锁

**如果偏向锁被关闭或者当前偏向锁已经已经被其他线程获取，那么这个时候如果有线程去抢占(竞争)同步锁时，锁会升级到轻量级锁。**

- 轻量级锁加锁

线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，**如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。**

- 轻量级锁解锁

轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。下图是两个线程同时争夺锁，导致锁膨胀的流程图。

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%2010.png](https://img.masaiqi.com/20200618223935.png)

**因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。**

## 4.3. 重量级锁

**多个线程竞争同一个锁的时候，在自旋到一定次数后依然无法获得锁，锁会升级到重量级锁。虚拟机会阻塞加锁失败的线程，并且在目标锁被释放的时候，唤醒这些线程。**

每一个JAVA对象都会与一个监视器monitor关联，我们可以把它理解成为一把锁，当一个线程想要执行
一段被synchronized修饰的同步方法或者代码块时，该线程得先获取到synchronized修饰的对象对应
的monitor。

monitorenter表示去获得一个对象监视器。monitorexit表示释放monitor监视器的所有权，使得其他
被阻塞的线程可以尝试去获得这个监视器。

**monitor依赖操作系统的MutexLock(互斥锁)来实现的,线程被阻塞后便进入内核(Linux)调度状态，这
个会导致系统在用户态与内核态之间来回切换，严重影响锁的性能。**

任意线程对Object(Object由synchronized保护)的访问，首先要获得Object的监视器。如果获取失
败，线程进入同步队列，线程状态变为BLOCKED。当访问Object的前驱(获得了锁的线程)释放了
锁，则该释放操作唤醒阻塞在同步队列中的线程，使其重新尝试对监视器的获取。

## 4.4. 各种锁的对比

《Java并发编程的艺术》一书中作以下表述：

![synchronized%204b56d0bfe1e84a57a0493bb4c2ccd38f/Untitled%2011.png](https://img.masaiqi.com/20200618223939.png)

# 5. 参考

- 《Java并发编程的艺术》