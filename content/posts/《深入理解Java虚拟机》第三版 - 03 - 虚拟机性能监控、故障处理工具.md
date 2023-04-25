---
title: "《深入理解Java虚拟机》第三版 - 03 - 虚拟机性能监控、故障处理工具"
date: 2022-09-08 19:40:33.436
draft: false
type: "post"
showTableOfContents: true
tags: ["Java","虚拟机"]
---

# 1. 基础故障处理工具

## 1.1. jps: 虚拟机进程状况工具

**JPS（JVM Process Status Tool），可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（LVMID，Local Virtual Machine Identifier）**

![Untitled](https://img.masaiqi.com/202209081939531.png)

jps命令格式：

`jps [options] [hostid]`

如果主类名称相同，比如图中的Application，就得通过其他JPS命令的参数来区分：

![Untitled](https://img.masaiqi.com/202209081939568.png)

## 1.2. jstat：虚拟机统计信息监视工具

**jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据。**

jstat命令格式为：

`jstat [ option vmid [interval[s|ms] [count]] ]`

- 如果是本地虚拟机进程，VMID与LVMID是一致的，如果是远程虚拟机进程，那VIMD的格式为：`[protocol:][//]lvmid[@hostname[:port]/servername]`
- interval和count代表查询间隔和次数，如果省略，默认只查询一次
- 

查询进程67027 GC情况，每250秒查询一次，查询20次：

![Untitled](https://img.masaiqi.com/202209081939590.png)

选项option代表用户希望查询的虚拟机信息，主要分为三类：类加载、垃圾收集、运行期编译状况。

![Untitled](https://img.masaiqi.com/202209081939616.png)

## 1.3. jinfo：Java配置信息工具

**jinfo（Configuration Info for Java）的作用是实时查看和调整虚拟机各项参数。**

**jps -v命令可以查看虚拟机启动时显式指定的参数列表，如果想知道未被显式指定的参数的系统默认值，可以使用jinfo的flag选项进行查询了**（如果只限于JDK6或以上版本的话，使用javaXX：+PrintFlagsFinal查看参数默认值也是一个很好的选择）。

jinfo命令格式：

`jinfo [option] pid`

查询进程67027的所有启动参数：

![Untitled](https://img.masaiqi.com/202209081939634.png)

## 1.4. jmap：Java内存映像工具

**jmap（Memory Map for Java）命令用于生成堆转储快照，一般称为heapdump或dump文件**。（如果不使用jmap命令，要想获取Java堆转储快照也还有一些比较“暴力”的手段：比如-XX：+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在内存溢出异常出现之后自动生成堆转储快照文件，通过XX：+HeapDumpOnCtrlBreak参数则可以使用[Ctrl]+[Break]键让虚拟机生成堆转储快照文件，又或者在Linux系统下通过Kill3命令发送进程退出信号“恐吓”一下虚拟机，也能顺利拿到堆转储快照。）

**jmap的作用并不仅仅是为了获取堆转储快照，它还可以查询finalize执行队列、Java堆和方法区的详细信息，如空间使用率、当前用的是哪种收集器等。**

jmap命令格式：

jmap [ option ] vmid

如果想要获取进程67027存活对象的dump文件：

![Untitled](https://img.masaiqi.com/202209081939650.png)

如果想要获取进程67027所有对象的dump文件：

![Untitled](https://img.masaiqi.com/202209081939666.png)

![Untitled](https://img.masaiqi.com/202209081939682.png)

## 1.5. jhat：虚拟机堆转储快照分析工具

**JDK提供jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆转储快照。jhat内置了一个微型的HTTP/Web服务器，生成堆转储快照的分析结果后，可以在浏览器中查看**。

**jhat不常用，因为大多数情况都会将jmap的dump文件拉取到本地，用更强大的工具进行分析。**

## 1.6. jstack：Java堆栈跟踪工具

**jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因**，如线程间死锁、死循环、请求外部资源导致的长时间挂起等，都是导致线程长时间停顿的常见原因。**线程出现停顿时通过jstack来查看各个线程的调用堆栈，就可以获知没有响应的线程到底在后台做些什么事情，或者等待着什么资源**。

jstack命令格式：

`jstack [option] vmid`

查看进程2433的线程信息：

![Untitled](https://img.masaiqi.com/202209081939707.png)

![Untitled](https://img.masaiqi.com/202209081939724.png)