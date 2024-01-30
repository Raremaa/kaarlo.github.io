---
title: "Wireshark网络协议分析 - Wireshark速览"
date: 2024-01-30 15:43:52.95
draft: false
type: "post"
showTableOfContents: true
tags: ["wireShark","网络协议"]
---

# 1. 版本与平台

[`Wireshark`](https://www.wireshark.org/)是一个开源的网络数据包分析器，本文基于`Windows11`平台，`Wireshark`使用的是4.2.0版本，有官方的中文支持。

![Untitled](https://masaiqi.oss-cn-hangzhou.aliyuncs.com/Untitled.png)

# 2. 快速上手

## 2.1. 选择网络接口进行捕获（Capture）

打开`Wireshark`第一步需要选择一个网络接口：

![Untitled](https://masaiqi.oss-cn-hangzhou.aliyuncs.com/Untitled%201.png)

这里可以根据实际情况选择一个接口，笔者这里是WLAN上网的环境，重点看一下两个接口：

- `WLAN`：无线局域网流量接口
- `adapter for loopback traffic capture`：回环流量接口

举2个例子。

*场景一：*

我本地启动了一台tcp服务器，它的地址端口是127.0.0.1:9820；同时，我在本地9821端口启动了一台tcp客户端连接这个服务器，他的流量路径是：

127.0.0.1:9821 → 127.0.0.1:9820

此时，流量无需经过外部网关转发，是一个纯粹的系统内部端口直连流量，这种情况下就应该使用`adapter for loopback traffic capture`接口进行捕获。

*场景二：*

如果我是本地电脑访问比如`https://masaiqi.com`这种公网的网站，需要通过WLAN接口访问路由器进而访问，应该使用`WLAN`接口进行捕获。

## 2.2. 以Ping命令为例进行抓包分析

我这里选择的是捕获WLAN接口，执行以下命令：

```bash
ping masaiqi.com
```

在WireShark中我们有以下结果：

![Untitled](https://masaiqi.oss-cn-hangzhou.aliyuncs.com/Untitled%202.png)

## 2.3. 设置合适的过滤表达式

WLAN接口是我们本地访问公网的接口，所有流量都将被拦截，我们首先需要通过过滤器去定位我们需要的流量。

我们知道，Ping命令主要包含两个过程，DNS Query + ICMP。

因此，可以看到在WireShark的过滤表达式我们输入了以下过滤条件，即DNS协议或者ICMP协议：

```bash
dns or ((_ws.col.protocol == "ICMP") )
```

**当表达式正确时，`Wireshark`会显示绿色，反之则是红色。**

## 2.4. 数据包详情

这部分是Wireshark的核心部分，以序号1167这行数据为例，点击后WireShark显示如下：

![Untitled](https://masaiqi.oss-cn-hangzhou.aliyuncs.com/Untitled%203.png)

主要分为两部分：

- 绿圈部分则为原始的二进制数据流，当然这里`Wireshark`采用了十六进制进行展示。
- 红色方框为原始二进制数据流的具体含义，Wireshark很贴心的帮我们“翻译”原始二进制数据流。当鼠标放在红圈的某一行数据时，`Wireshark`会在绿圈中用蓝色背景色提示你这里对应原始数据流的哪一部分数据。

点开“Domain Name System”：

![Untitled](https://masaiqi.oss-cn-hangzhou.aliyuncs.com/Untitled%204.png)

***根据笔者的观察，这里分为两种字段：***

- 直接展示的字段，比如`Flags`，直接值就是0x0100，和原始数据流一一对应。
- 用中括号包裹的字段，比如`[Name Length]`，这种一般是字节流没有直接表示，而是`Wireshark`根据当前上下文（可能不止当前的包）推断出来的。

数据包详情中有部分数据Wireshark会帮我们显示在Info列中，比如这里我们从封包内容读出是`DNS`查询`masaiqi.com`，数据包的Info列中也有相关内容展示。

## 2.5. TCP/IP 四层模型

封包详情中这部分数据刚好对应TCP/IP四层模型中的模型层次（原谅笔者草率的图）

![Untitled](https://masaiqi.oss-cn-hangzhou.aliyuncs.com/Untitled%205.png)

`TCP/IP 四层模型`，是分析`Wireshark`数据帧（数据包）的基础知识。

关于网络的框架模型普遍存在两套理解，即OSI七层模型与TCP/IP四层模型，本质上只是考虑问题的维度不一样，二者的关系如下：

![Untitled](https://masaiqi.oss-cn-hangzhou.aliyuncs.com/Untitled%206.png)

对于TCP/IP四层模型来说：

- **Application（应用层）**：用户直接与之交互的层，负责处理具体的应用程序细节。包括HTTP等协议（本例中使用域名查询服务）
- **Transport（传输层）**：负责在源点和目的地之间提供端到端的通信。包括TCP协议，UDP协议。（本例中使用`UDP`协议）
- **Internet（网络层，网络互连层）**：负责在网络中传输数据包。包括互联网协议（IP）。（本例中的`IP`协议，源ip与目标IP）
- **Network（网络接口层）**：负责在以太网、WiFi 这样的底层网络上发送原始数据包，工作在网卡这个层次，使用 MAC 地址来标识网络上的设备。（本例中是网络设备的`MAC`地址，源设备`MAC`地址与目标`MAC`地址）

这里仅快速了解一下`TCP/IP四层模型`的知识，不结合实例来说就是死记硬背了，后续会结合具体的封包内容，按协议不同单独分析。