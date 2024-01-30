---
title: "Wireshark网络协议分析 - TCP协议"
date: 2024-01-30T17:43:59+08:00
draft: false
type: "post"
showTableOfContents: true
tags: ["wireShark","网络协议"]
---

# 1. 基础

根据TCP/IP四层模型，在TCP场景下如下图：

![Untitled](https://img.masaiqi.com/202401301818705.png)

其中，TCP头的信息：

![Untitled](https://img.masaiqi.com/202401301818962.png)

# 2. 实战

### 2.1. 用Go写一个简单的TCP服务器与客户端

要分析`TCP`协议，尽管可以直接使用一个`http`的访问进行抓包，但是`http`的过程中又涉及`DNS`，`ARP`等过程混淆我们，为了纯粹的分析`TCP`协议，我们这里使用`Golang`写了一个简单的9830端口的`TCP`服务器与客户端，源代码简单展示如下：

*服务端：*

```go
package server

import (
	"fmt"
	"net"
	"os"
	"strings"
	"test/util"
)

func StartTCPServer(c chan<- string) {
	listener, err := net.Listen("tcp", "localhost:9830")
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}

	defer listener.Close()
	c <- "ready"

	// 等待连接
	conn, err := listener.Accept()
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}
	// 处理连接
	handleRequest(conn)
}

func handleRequest(conn net.Conn) {
	defer conn.Close()

	var res strings.Builder
	buf := make([]byte, 1024)
	for {
		n, err := conn.Read(buf)
		if err != nil {
			break
		}
		res.Write(buf[:n])
	}

	fmt.Print("Received client message, size:", res.Len())
}
```

*客户端：*

```go
package client

import (
	"fmt"
	"net"
	"os"
	"strings"
	"test/util"
)

func StartTCPClient() {
	conn, err := net.Dial("tcp", "localhost:9830")
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}

	defer conn.Close()

	// 发送200kb消息
	largeMessage := strings.Repeat("A", 200*1024)
	_, err = conn.Write([]byte(largeMessage))
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}
	fmt.Println("Sent message to server!")
}
```

*执行入口：*

```go
package main

import (
	"fmt"
	"sync"
	c "test/internal/client"
	s "test/internal/server"
)

// main wireshark filter express: tcp.port==9830
func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	serverReady := make(chan string, 1)

	go func() {
		s.StartTCPServer(serverReady)
		wg.Done()
	}()

	go func() {
		<-serverReady
		c.StartTCPClient()
		wg.Done()
	}()

	wg.Wait()
}
```

编译执行，控制台输出如下：

![Untitled](https://img.masaiqi.com/202401301818971.png)

### 2.2. Wireshark抓包分析

由于我们这里`TCP`的客户端和服务端都是面向`localhost` ，使用`adapter for loopback traffic capture`接口捕获回环流量，过滤器过滤tcp端口9830即可：

```bash
tcp.port==9830
```

![Untitled](https://img.masaiqi.com/202401301818404.png)

**需要注意，即使代码和我一样，最终抓包的数据依然会有差距，因为`MTU`，`window size`等参数会因为`OS`的策略不同有差异，求同存异抓共性。**

***大体分为三部分：***

- 协议连接建立与确认：即包号92~96部分，图中具体又分为两部分：
    - 包号92~93中的Source和Destination都是::1，这个地址是IPV6的地址，类似于IPV4中的127.0.0.1。TCP客户端先尝试了连接IPV6的9830端口，被服务器中断（RST）。
    - 包号94~96尝试了尝试了IPV4的9803端口，这一次得到了服务器的正确回应，开始了**`TCP`协议中的“三次握手”，此时建立`TCP`连接**
- 具体的业务数据交互：即包号97~103
- 协议连接断开与确认：即包号104~107，也就是**`TCP`协议中的“四次挥手”，此时释放`TCP`连接**

***点开包号96，我们查看其Network层的数据情况：***

![Untitled](https://img.masaiqi.com/202401301818638.png)

`Source Port`之类的字段比较直观易于理解，下面我们重点看一下一些不太直观的字段代表的数据含义。

### 2.3. 限制数据包的大小——MSS与MTU

**网络对包的大小是存在限制的，这部分是在建立TCP连接的“三次握手”中进行声明**，即：

- `MSS`（Maximum Segment Size，最大段大小）
- `MTU`（Maximum Transmission Unit，最大传输单元）：`MSS`+ `TCP`头长度 + `IP`头长度即是`MTU`

![Untitled](https://img.masaiqi.com/202401301818910.png)

在“三次握手中”，包号94中，客户端向服务端声明了`MSS=65495` ，可以计算`MTU`（**单位为Byte，一般Wireshark中出现的数字单位都是Byte**）：

`MTU = 65495 + 20(TCP头) + 20(IP头) = 65535`

同理，包号95中，服务端在给客户端的`ACK`包中也声明了自己的`MSS`，也可以计算出`MTU`。

**客户端和服务端各自有MTU，实际的传输包的大小是由客户端与服务端2个`MTU`中较小的数值去决定的。**

### 2.4. 保证TCP的有序传输——Seq，Len与Ack

**`TCP`提供的是有序传输**，每个数据段都要标上一个序号。

在实际场景中，**`TCP`传输数据时并不能严格保证有序，总会出现乱序的情况，需要根据序号进行排序**。

- **`Len`：length，数据长度**
- **`Seq`：Sequence Number，序号，即上一个数据段`Seq` + `Len`，发送方和接收方都需各自维护Seq状态**
- **`Ack`：Acknowledge Number，确认号，待接受的数据序号,发送方和接收方都需各自维护`Ack`状态**

以包号97~99为例：

![Untitled](https://img.masaiqi.com/202401301818575.png)

- 97号：客户端发送数据，声明自己的`Seq`=1，`Len`=65495
- 98号：服务端声明自己的`Seq`=1，`Len`=0，同时计算`Ack`=65496，即发送方的`Seq` + `Len`
- 99号：客户端发送数据，声明自己的`Seq`=65496，`Len`=65495，`ACK`为1，即上一步中服务端的`Seq` + `Len`

可以推断出：

- `Seq`和`Len`主要表达的是作为发送方当前的数据序号和长度
- `Ack`主要表达的是作为接收方的接收到的数据序号和长度的和
- `TCP`通信中，一个客户端/服务端既有发送的场景，也有接受的场景，因此`Seq`，和`Ack`都各自需要单独维护，`Len`则是实际的数据长度。

### 2.5. TCP头标志位——URG，ACK，PSH，RST，SYN，FIN

`TCP`头包含一系列的标识位（也称作控制位或标志位），它们用于控制和管理`TCP`连接的不同方面。每个标识位都占用`TCP`头部中的1个比特。

Wireshark不仅帮我们“翻译”了当前封包的状态，也把TCP头的所有状态位都做了提示。

![Untitled](https://img.masaiqi.com/202401301818487.png)

- **URG（紧急）**：表示紧急指针字段（Urgent Pointer）有效，用于指示紧急数据。
- **ACK（确认）**：表示确认字段（Acknowledgment Field）有效。几乎所有的TCP报文，除了最初的SYN报文外，都会设置这个标志。
- **PSH（推送）**：提示接收端应该立即将这个报文交给应用层，而不是等待缓冲区满。
- **RST（重置）**：用于重置错误的连接，或拒绝非法的报文或打开的连接。
- **SYN（同步）**：在建立连接时用来同步序列号。当一个连接开始时，用来初始化序列号并开始连接建立。
- **FIN（结束）**：用于释放一个连接。当数据传输结束时，发送方用它来告诉接收方它已经结束发送数据。

### 2.6. TCP连接建立——三次握手

包号94~96行则为`TCP`的“三次握手”行为，发生在`TCP`连接建立之初：

![Untitled](https://img.masaiqi.com/202401301819090.png)

**“三次握手”主要分为3步：**

1. **客户端 → 服务端：【SYN】申请建立连接，声明初始`Seq`**
2. **服务端 → 客户端：【SYN，ACK】收到连接请求，声明初始`Seq`，声明`Ack`=客户端声明的初始`Seq` + 1**
3. **客户端 → 服务端：【ACK】收到确认请求，声明`Ack`=服务端初始`Seq` + 1**

### 2.7. TCP连接释放——四次挥手

包号104~107行则为`TCP`的“四次挥手”行为，发生在TCP连接需要中断，释放连接时：

![Untitled](https://img.masaiqi.com/202401301819819.png)

**“四次挥手”**主要分为4步，**发起方可能是客户端，也可能是服务端：**

1. 发起方发起释放连接请求：【FIN，ACK】，声明`Seq`和`Ack`
2. 接受方返回确认收到：【ACK】，声明`Ack`=第1.步中发起方的`Seq` + 1
3. 接收方发起释放连接请求：【FIN，ACK】，声明`Seq`和`Ack`（与上一步一样）
4. 发起方返回确认收到：【ACK】，声明`Ack`=第3.步中接收方的`Seq` + 1

### 2.8. 大包拆分——累计确认，TCP窗口

- 我们`TCP`客户端代码中发送了一个200kb的字符串给服务端，之前分析过我们的`MTU`限制在65535b，不算`TCP`头和`IP`头的`MSS`为65495b。**我们实际无法在一个数据帧中发送全部数据，需要按照MTU拆分为多个数据帧进行发送。**
- **多个数据帧的发送不一定每一条都需要等待接收方回复ACK，可以发送若干条数据帧之后，接收方回复一条ACK即可，这是TCP的累计确认。**
- `TCP`连接的一方连续发送的数据帧数是存在限制的，否则无限发送数据帧，另一方处理速率赶不上就会遇到`Back Pressure（背压）`的情况，这个限制则是`TCP Window(TCP窗口)`，即`Wireshark`中的`Win`的数值：

![Untitled](https://img.masaiqi.com/202401301819447.png)

1. 包号95是三次握手的第二次，**win=65535，这是声明自己的接受窗口大小是65535**，在我们的例子中是服务端声明自己的接受窗口大小是65535，这是协商的初始窗口大小。
2. 客户端按照`MTU`/`MSS`要求拆分数据帧
3. 包号97，客户端向服务端发送了`Len`=65495的数据
4. 包号98，服务端向客户端发送`ACK`，调整了自己的TCP`窗口`，声明`Win`=2161152。这里如果窗口大小为零，发送方将停止发送数据，并等待下一个窗口更新。也就是说，**窗口大小是动态调整的。**
5. 客户端获知服务端的窗口变大了，因此自己可以连续发送多个数据帧
6. 包号99~101，客户端连续向服务端发送了3个数据帧，无需等待服务端回复`ACK`，`Seq`按序增加，由于服务端没有回复`ACK`，这3帧数据的Ack都一样为1。
7. 包号102，服务端向客户端发送了`ACK`，`Ack`=204801，刚好是包号101的`Seq` + `len`，这里实际上代表当前客户端已经收到了包号99~101的内容，这被称为`TCP`的累计确认。

窗口大小在`TCP`协议设计之初只留了2byte，这个可以在Wireshark中看到：

![Untitled](https://img.masaiqi.com/202401301819391.png)

后续为了提升Wireshark的长度，在TCP“三次握手”时的`Options`中有`Window scale` ：

![Untitled](https://img.masaiqi.com/202401301819538.png)

最终窗口大小计算公式 = Window的数值 8442 * 2的window scale次方

### 2.9. 动态调整TCP窗口——从慢启动到阻塞避免

在上一部分大包拆分中，可以看到TCP窗口是会动态调整的，这是有一个具体的调整策略的。

1. **连接刚建立，发送方对网络情况一无所知，需要定一个初始值**，可以是几个MSS大小。
2. 连接初期，由于发生拥塞概率很低，如果发出去的包都得到确认，可以尝试增大阻塞窗口，**RFC建议是每收到n个确认，阻塞窗口就增大n个MSS，这个过程需要慢慢增长，称为慢启动过程**。
3. 持续一段时间后，窗口大小达到一个较大的值，发生拥塞的概率变大，不能继续使用慢启动的算法，需要再缓慢一点，**RFC建议每个往返时间增加1个MSS，这个过程称为拥塞避免。**

![Untitled](https://img.masaiqi.com/202401301819445.png)

### 2.10. 丢包问题——RTO与快速重传

`TCP`连接过程中有可能出现丢包的情况。

**对于发送方来说，如果发出去的包不像往常一样得到确认，有可能是网络延迟导致，发送方会等待一段时间再去判断，如果一直收不到，就会判定包已丢失，进行重传。这个过程称为超时重传，从发出原始包刀重传该包的时间段称为`RTO`：**

![Untitled](https://img.masaiqi.com/202401301819249.png)

**重传之后，`TCP`窗口也会被调整，会先降到1个`MSS`，之后会进入慢启动过程。不过这次从慢启动到拥塞的临界窗口值有了参考依据**，RFC5681建议应该设置为拥塞时没被确认的数据量的1/2，不小于2个`MSS`：

![Untitled](https://img.masaiqi.com/202401301819690.png)

RTO的过程对性能影响是比较大的：

1. **RTO阶段等待重传不能传数据，浪费处理时间**
2. **RTO之后的TCP窗口急剧减小，传输比之前慢**

与RTO不同，**如果拥塞很轻微，发生部分丢包，发送方还是能接收到部分Ack包，Ack会有期望Seq号，通过这个期望Seq号，发送方会意识到发生包丢失，当发送方收到3个及以上Dup Ack(重复确认)时会立刻重传，这个过程称为快速重传**。

这里之所以**需要凑齐3个及以上Dup Ack的原因是实际场景中包可能会乱序，Ack好像表现出一个缺失的包，但是这个包并没有丢失，而是由于乱序还在数据传输路上，可能很快就会传输过来，因此设置一个数量要求一定程度避免乱序导致快速重传**：

![Untitled](https://img.masaiqi.com/202401301819471.png)

**快速重传的过程对性能影响是比较小的，因为依然有部分包能够被发送和接收到，说明拥塞并不严重，因此只需传慢一点即可，无需突然大幅度TCP窗口重新慢启动**。RFC5681建议重新设置临时窗口为拥塞时没被确认数据的1/2，不小于2个MSS，拥塞窗口设置为临界窗口+3个MSS

![Untitled](https://img.masaiqi.com/202401301819842.png)

### 2.11. 减轻网络负担可能降低性能——延迟确认与Nagle算法

- **延迟确认：如果收到一个包后暂时没什么数据要发给对方，那就延迟一段时间再确认；假设这段时间内恰好有数据要发出，则确认信息和数据可以在一个包里发。**延迟确认**减少了部分确认包，减轻了网络负担，但有时候会影响性能，带来传输数据延迟。**
- **Nagle算法：在发出去的数据还没有被确认之前，如果有小数据生成，就把小数据收集起来凑满一个MSS或者等收到确认后再发送。**

# 3. 参考资料

- **林沛满 -《Wireshark网络分析就这么简单》**
- **刘超 ——《趣谈网络协议》**