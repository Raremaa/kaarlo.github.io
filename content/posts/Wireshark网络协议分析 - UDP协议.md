---
title: "Wireshark网络协议分析 - UDP协议"
date: 2024-01-30T17:49:59+08:00
draft: false
type: "post"
showTableOfContents: true
tags: ["wireShark","网络协议"]
---

# 1. 基础

`UDP`包的数据结构：

![Untitled](https://img.masaiqi.com/202401301820404.png)

# 2. 实战

### 2.1. 用Go写一个简单的UDP服务器与客户端

我们这里使用`Golang`写了一个简单的9830端口的`UDP`服务器与客户端，源代码简单展示如下：

*服务端：*

```go
package server

import (
	"fmt"
	"net"
	"os"
	"test/util"
)

func StartUDPServer(c chan<- string) {
	addr := "localhost:9829"
	udpAddr, err := net.ResolveUDPAddr("udp", addr)
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}

	conn, err := net.ListenUDP("udp", udpAddr)
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}
	defer conn.Close()
	c <- "ready"
	fmt.Println("UDP server listening on", addr)

	buffer := make([]byte, 1024)

	for {
		n, clientAddr, _ := conn.ReadFromUDP(buffer)
		if n > 0 {
			fmt.Printf("Received '%s' from %s\n", string(buffer[:n]), clientAddr)
			return
		}
	}
}
```

*客户端：*

```go
package client

import (
	"fmt"
	"net"
	"os"
	"test/util"
)

func StartUDPClient() {
	serverAddr := "localhost:9829"
	udpAddr, err := net.ResolveUDPAddr("udp", serverAddr)
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}

	conn, err := net.DialUDP("udp", nil, udpAddr)
	if err != nil {
		util.HandleError(err)
		os.Exit(1)
	}
	defer conn.Close()

	message := []byte("Hello UDP server!")
	_, err = conn.Write(message)
	if err != nil {
		fmt.Println("Error writing to UDP:", err)
		os.Exit(1)
	}

	fmt.Println("Message sent to server:", string(message))
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

// main wireshark filter express: udp.port==9829
func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	serverReady := make(chan string, 1)

	go func() {
		s.StartUDPServer(serverReady)
		wg.Done()
	}()

	go func() {
		<-serverReady
		c.StartUDPClient()
		wg.Done()
	}()

	wg.Wait()
	fmt.Println("done!")
}
```

编译执行，控制台输出如下：

![Untitled](https://img.masaiqi.com/202401301821680.png)

### 2.2. Wireshark抓包分析

由于我们这里`UDP`客户端和服务端都是面向`localhost` ，使用`adapter for loopback traffic capture`接口捕获回环流量，过滤器过滤tcp端口9829即可：

```bash
udp.port==9829
```

![Untitled](https://img.masaiqi.com/202401301821270.png)

可以看到，`UDP`协议的过程是比较简单的，无需`TCP`一样的“三次握手”操作，仅需直接对着监听端口发送数据，接收方接受数据即可。

在`Wireshark`的`Transport`分析中，我们可以看到上述的UDP包头信息：

![Untitled](https://img.masaiqi.com/202401301820391.png)

其中包括了源和目标端口地址，长度，校验和和数据payload信息。

# 3. UDP与TCP的区别

- **本质的区别是`TCP` 是有状态的，面向连接的，`UDP`是面向无连接的。`TCP`会三次握手，维护客户端和服务端的连接，建立一定的数据结构来维护双方交互的状态，`UDP`则不会**。
- 剩下的区别都是基于这个本质特性不一样衍生出的应用特性，比如：
    - `**TCP`提供可靠交付。通过 TCP 连接传输的数据，无差错、不丢失、不重复、有序。**
    - `**UDP`不保证不丢失，不保证按顺序到达**。
