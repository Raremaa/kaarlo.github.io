---
title: "博客被挖矿程序xmrig入侵以及我的解决方案(docker安全)"
date: 2019-09-03 00:00:00.0
draft: false
type: "post"
showTableOfContents: true
tags: ["misc"]
---

> 文章记录了一次由于Docker配置的漏洞导致的挖矿程序入侵的发现过程以及解决方案。由于本人时间有限，没有时间去深度学习linux的相关运维知识，所以解决方案可能是比较浅显的，还望您在阅读本文前知悉。

## 前言

2019年9月2日凌晨，我正准备关闭电脑睡觉，突然收到阿里云的一条短信，说是服务器被挖矿软件入侵了，一开始我是不相信的，就我这草履虫一样的网站，应该没有任何会来才对啊，后来查看了一下阿里云后台，我确定我的服务器确实被矿总入侵了。  

**阿里安全中心给出的结论**
**![](https://img.masaiqi.com/2019-09-03-010818.png)**
**![](https://img.masaiqi.com/2019-09-03-011125.png)**
**控制台CPU占用**  
我的服务器只挂载了一个Docker和Nginx应该没有那么大的资源占用，很明显这是异常占用了。
![](https://img.masaiqi.com/2019-09-03-011152.png)

## 原因分析与解决方案

### 寻找原因

#### 查找挖矿进程

````
top -H
````

![](https://img.masaiqi.com/2019-09-03-020953.png)

从图上我们可以找到CPU占用97.7的进行名为`xmrig`(甚至都不伪装一下进程名)

#### 查找挖矿程序文件位置
````shell
root@xxx:/# find / -name '*xmrig*'
/var/lib/docker/overlay2/7e28a5f8f394ab1242ed26d38f60c0f5023f02ddb307276fbf71bcbbc43e9399/merged/xmrig
/var/lib/docker/overlay2/d5e73ad017bf13ca294b65ba6f8f07e17e334b76e8ad5f43a10bd7737791bd6f/diff/xmrig
````

笔者发现这两个挖矿程序都是在docker的路径下，初步判断应该是docker的配置导致的

笔者想起来前些日子为了idea能够一键部署docker偷了一个懒：

**开放了无CA认证的Docker远程访问**

矿总应该是通过这个渠道连接到主机植入了挖矿程序。看到这个路径，笔者考虑是不是由于之前开放了远程访问，所以矿总部署了挖矿镜像？
````
docker ps -a
````
![](https://img.masaiqi.com/2019-09-03-031614.png)
果然如此。

#### 我们尝试查杀挖矿程序

- 首先，移除docker容器，删除相关镜像
````
docker rm -fv efd4021b4e66(容器id)
docker rmi tanchao2014/mytest
````

- 之前我们发现了是因为Docker的无认证的远程访问开启导致的入侵，所以这里要做的是关闭Docker无认证远程访问权限，避免下次被入侵
	1. 先关闭Docker
	````
	systemctl stop  docker
	````
	
	2. 配置docker远程访问
	````
	vi /lib/systemd/system/docker.service
	````
	之前的配置：
  ````shell
  [Service]
  Type=notify
  # the default is not to use systemd for cgroups because the delegate issues still
  # exists and systemd currently does not support the cgroup feature set required
  # for containers run by docker
  ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock
  ExecReload=/bin/kill -s HUP $MAINPID
  TimeoutSec=0
	RestartSec=2
	Restart=always
	````
	移除ExecStart中的`-H tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock`
	现在就有如下配置：
	````shell
	[Service]
  Type=notify
  # the default is not to use systemd for cgroups because the delegate issues still
  # exists and systemd currently does not support the cgroup feature set required
  # for containers run by docker
  ExecStart=/usr/bin/dockerd -H fd://
  ExecReload=/bin/kill -s HUP $MAINPID
  TimeoutSec=0
  RestartSec=2
  Restart=always
	````
	
	3. 重新读取配置，启动docker
	````shell
	systemctl daemon-reload
	systemctl start docker
	````
	
- `top -H`查看CPU占用最高的进程，发现恢复正常，整机cpu占用百分之7点几，符合网站预期。

## 教训与经验

这是我第一次亲身感受服务器被入侵，好在矿总大哥手下留情，只是部署了一个挖矿的Docker容器，没有在服务器其他地方植入矿机。虽然服务器成为肉鸡令人难过，可是这次的教训却让我以后再部署实施时更加注意服务器安全问题，让我意识到之前的配置存在的问题而且没有造成很严重的后果，总体是收获大于损失(几乎没损失哈哈)。

**如果你也之前看到过idea一键部署Docker的教程，千万记得要配置CA认证的远程访问，不要偷懒，不然你就是下一个矿机**

