---
title: "RocketMQ学习与部署Docker记录(后续更新)"
date: 2019-09-10 00:00:00.0
draft: false
type: "post"
showTableOfContents: true
tags: ["RocketMQ","MQ"]
---

---
layout:     post
title:      "RocketMQ学习与部署Docker记录(后续更新)"
date:       2019-09-10
author:     "马赛琦"
tags:

    - 消息队列
        - RocketMQ

    - java
---

## 1. RocketMQ简介

**RocketMQ**是一个分布式消息和流数据平台(消息中间件)，具有低延迟、高性能、高可靠性、万亿级容量和灵活的可扩展性。RocketMQ是2012年[阿里巴巴](https://zh.wikipedia.org/wiki/阿里巴巴集团)开源的第三代分布式消息中间件，2016年11月21日，阿里巴巴向[Apache软件基金会](https://zh.wikipedia.org/wiki/Apache软件基金会)捐赠了RocketMQ；第二年2月20日，Apache软件基金会宣布Apache RocketMQ成为顶级项目。  

**阅读本文前，您应该同意以下前提：**

- **本文并不会讲述RocketMQ的原理，仅讨论RocketMQ的使用，原理会在后续进行更新**

- **假设您已经知道消息队列是什么，解决了什么问题**

- **假设您已经掌握Git的基本使用**

- **假设您已经掌握Docker的基本使用(非必须)**

- **假设您已经掌握MAVEN的基本使用**

整体架构图([原图出处](https://blog.csdn.net/iie_libi/article/details/54236502))：

![](https://img.masaiqi.com/2019-09-10-075436.jpg)

**本文将以目前(2019年9月10日)最新版RocketMQ版本-4.5.2，以官方文档提供的指导进行陈述。**

## 2. RocketMQ体验

### 2.1. 安装RocketMQ(服务端)

这里以Docker安装为例，以[官方github](https://github.com/apache/rocketmq-docker)作为参考

#### 2.1.1. 拉取`RocketMQ-docker` 代码(git或wget)

````shell
git clone https://github.com/apache/rocketmq-docker.git
//或者用下面的方式，需要解压
wget https://github.com/apache/rocketmq-docker/archive/master.zip
````

#### 2.1.2. 构建RockerMQ-docker镜像(alpine环境为例)

````shell
cd image-build
sh build-image.sh 4.5.2 alpine
````

笔者这里使用的是当前最新版本4.5.2

等待程序运行结束后，我们查看镜像：

````
root@XXX:/home/masaiqi/rocketMQ/rocketmq-docker-master/image-build# docker images
REPOSITORY             TAG                 IMAGE ID            CREATED              SIZE
rocketmqinc/rocketmq   4.5.2-alpine        a580c1ac67af        About a minute ago   143MB
````

可以看到，我们镜像已经构建成功

**你可能会在运行sh脚本的时候和笔者出现一些错误，笔者这里列出两种解决方式（亲测可用**）：

- windows和linux幻想下换行符不一致，笔者在linux环境下是利用了`tofrodos`解决的，`tofrodos`包括两部分：todos（相当于unix2dos），fromdos（相当于dos2unix）
````shell
sudo apt-get install tofrodos 
fromdos XXX.sh
````
- 给sh文件增加读写执行权限
````shell
chmod u+x XXX.sh
````
#### 2.1.3. 构建RocketMQ容器实例(alpine环境为例)

````shell
sh stage.sh 4.5.2
cd /stages/4.5.2/templates
./play-docker.sh alpine
````

程序运行结束后，我们查看容器：

````
root@XXX:/home/masaiqi/rocketMQ/rocketmq-docker-master/stages/4.5.2/templates# docker ps -a
CONTAINER ID        IMAGE                               COMMAND                  CREATED             STATUS              PORTS                                                            NAMES
2133aeea4d26        rocketmqinc/rocketmq:4.5.2-alpine   "sh mqbroker"            13 seconds ago      Up 11 seconds       9876/tcp, 10909/tcp, 10911-10912/tcp                             rmqbroker
696818d53f86        rocketmqinc/rocketmq:4.5.2-alpine   "sh mqnamesrv"           14 seconds ago      Up 12 seconds       9876/tcp, 10909/tcp, 10911-10912/tcp                             rmqnamesrv
````

可以看到，容器已经正确创建并启动

#### 2.1.4. 正确启动却无法访问坑点

到这里其实还没有结束，有一个坑点需要注意：

**启动broker时，RocketMQ会指定为内网地址，使用的是172.17.0.3。会导致外网生产者无法连接到broker**

**启动脚本没有指定端口映射**

解决方案:

删除之前创建的容器

````shell
docker rm -fv [容器id]
````

修改之前的启动脚本`play-docker.sh`

````shell
vi play-docker.sh
````

找到如下docker命令语句，添加-c /home/rocketmq/rocketmq-4.5.2/conf参数指定配置文件(不添加-c不会读取conf)，这里要注意参数要加在`sh xxx`的后面，这个参数是这个脚本的参数，不是docker命令的参数

同时，分别指定端口号(笔者这里之前没有指定端口号，抓取端口发现相关端口都没有进程)

````shell
start_namesrv_broker()
{
    TAG_SUFFIX=$1
    # Start nameserver
    docker run -d -p 9876:9876 -v `pwd`/data/namesrv/logs:/home/rocketmq/logs --name rmqnamesrv rocketmqinc/rocketmq:4.5.2${TAG_SUFFIX} sh mqnamesrv -c /home/rocketmq/rocketmq-4.5.2/conf/broker.conf
    # Start Broker
    docker run -d -p 10911:10911 -p 10909:10909 -v `pwd`/data/broker/logs:/home/rocketmq/logs -v `pwd`/data/broker/store:/home/rocketmq/store --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" rocketmqinc/rocketmq:4.5.2${TAG_SUFFIX} sh mqbroker -c /home/rocketmq/rocketmq-4.5.2/conf/broker.conf
}
````

执行脚本，并进入broker容器中修改配置文件

````shell
./play-docker.sh alpine
docker exec -it [容器id] bash
````

进入容器，修改配置文件

````shell
vi ../conf/broker.conf
````

加入以下配置

````shell
brokerIP1 = [实际ip]
````

重启容器

````shell
docker restart [容器id]
````

### 2.2. 安装图形化管理程序

拉取镜像

````shell
docker pull styletang/rocketmq-console-ng
````

启动容器实例

````shell
docker run -d -e "JAVA_OPTS=-Drocketmq.namesrv.addr=[外部id]:端口 -Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8080:8080 -t styletang/rocketmq-console-ng
````

访问`ip:8080`可以看到我们的rocketMQ控制台

![](https://img.masaiqi.com/2019-09-12-040144.png)

### 2.3. Simple Example

新建父子MAVEN项目，父项目`rocketMQ-study`，子项目`simple-example`

父POM文件如下：

````xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.masaiqi</groupId>
    <artifactId>rocketMQ-study</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>

    <modules>
        <module>simple-example</module>
    </modules>

    <name>RockerMQ学习记录</name>

    <dependencies>
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-client</artifactId>
            <version>4.5.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <release>11</release>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
````

子POM文件如下：

````xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>rocketMQ-study</artifactId>
        <groupId>com.masaiqi</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <modelVersion>4.0.0</modelVersion>

    <artifactId>simple-example</artifactId>

    <name>RockerMQ学习记录::Simple-Example</name>

</project>
````

在子项目`simple-example`中类`SyncProducer`，用来同步发送消息

>Reliable synchronous transmission is used in extensive scenes, such as
>important notification messages, SMS notification, SMS marketing system, etc..
>
>大概意思就是，同步发送消息被广泛应用于各种场景，比如重要的消息通知，短信，短信营销系统等

````java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

/**
 * Producer
 * <p>
 * Send Messages Synchronously
 * Reliable synchronous transmission is used in extensive scenes,
 * such as important notification messages, SMS notification, SMS marketing system, etc..
 *
 * @author sq.ma
 * @date 2019/9/10 下午6:12
 */
public class SyncProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new
                DefaultMQProducer("msg_send_group");
        // Specify name server addresses.
        producer.setNamesrvAddr("你的ip:端口");
        //Launch the instance.
        producer.start();
        for (int i = 0; i < 100; i++) {
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTest",
                    "TagA",
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)
            );
            //Call send message to deliver message to one of brokers.
            SendResult sendResult = producer.send(msg);
            System.out.printf("%s%n", sendResult);
        }
        //Shut down once the producer instance is not longer in use.
        producer.shutdown();
    }
}
````

运行主函数，输出如下：

````
SendResult [sendStatus=SEND_OK, msgId=C0A802643E0D2C13DA153BB8C26A0000, offsetMsgId=743E2D7000002A9F0000000000015D6A, messageQueue=MessageQueue [topic=TopicTest, brokerName=broker-a, queueId=1], queueOffset=63]
...
````

这样消息就发送成功了，已经推送并存储到指定的`broker`中
下面新建`Consumer`类，用来消费(接受)消息

````java
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.client.exception.MQClientException;

/**
 * Consumer
 *
 * @author sq.ma
 * @date 2019/9/10 下午6:30
 */
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {

        // Instantiate with specified consumer group name.
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("msg_receive_group");

        // Specify name server addresses.
        consumer.setNamesrvAddr("你的ip:端口");

        // Subscribe one more more topics to consume.
        consumer.subscribe("TopicTest", "*");
        // Register callback to execute on arrival of messages fetched from brokers.
        consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
            System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        });

        //Launch the consumer instance.
        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
````

可以看到控制台打印出了我们之前发送的语句

````
ConsumeMessageThread_2 Receive New Messages: [MessageExt [queueId=7, storeSize=178, queueOffset=62, sysFlag=0, bornTimestamp=1568269164235, bornHost=/125.118.71.152:31579, storeTimestamp=1568269164245, storeHost=/ip:端口, msgId=743E2D7000002A9F0000000000016196, commitLogOffset=90518, bodyCRC=1307562618, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=74, CONSUME_START_TIME=1568269977672, UNIQ_KEY=C0A802643E0D2C13DA153BB8C2CB0006, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 54], transactionId='null'}]] 
````

你可以尝试将body中的byte数组转化为字符串，与之前发送的消息一致。

除此以外，官网还提供了几种发送消息方式，代码比较简单，直接上代码：

`AsyncProducer`用来异步发送消息

> Asynchronous transmission is generally used in response time sensitive business scenarios.
>
> 大概意思就是，异步发送通常被用于一些需要敏捷(快速)相应的业务场景

````java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendCallback;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

/**
 * @author sq.ma
 * @date 2019/9/12 下午2:35
 */
public class AsyncProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // Specify name server addresses.
        producer.setNamesrvAddr("localhost:9876");
        //Launch the instance.
        producer.start();
        producer.setRetryTimesWhenSendAsyncFailed(0);
        for (int i = 0; i < 100; i++) {
            final int index = i;
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTest",
                    "TagA",
                    "OrderID188",
                    "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
            producer.send(msg, new SendCallback() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    System.out.printf("%-10d OK %s %n", index,
                            sendResult.getMsgId());
                }
                @Override
                public void onException(Throwable e) {
                    System.out.printf("%-10d Exception %s %n", index, e);
                    e.printStackTrace();
                }
            });
        }
        //Shut down once the producer instance is not longer in use.
        producer.shutdown();
    }
}
````

`OnewayProducer`用来单步发送消息，特点为只负责发送消息，不等待服务器回应且没有回调函数触发，即只发送请求不等待应答。此方式发送消息的过程耗时非常短，一般在微秒级别。

> One-way transmission is used for cases requiring moderate reliability,
> such as log collection.
>
> 大概意思就是单步发送消息被用于不需要太高的稳定性的场景，比如日志收集

````java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

/**
 * @author sq.ma
 * @date 2019/9/12 下午2:40
 */
public class OnewayProducer {
    public static void main(String[] args) throws Exception{
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // Specify name server addresses.
        producer.setNamesrvAddr("localhost:9876");
        //Launch the instance.
        producer.start();
        for (int i = 0; i < 100; i++) {
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTest" /* Topic */,
                    "TagA" /* Tag */,
                    ("Hello RocketMQ " +
                            i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            //Call send message to deliver message to one of brokers.
            producer.sendOneway(msg);

        }
        //Shut down once the producer instance is not longer in use.
        producer.shutdown();
    }
}
````

## 3. 代码地址

[我的github](https://github.com/Raremaa/RocketMQ-study)

## 4. 参考资料

[RocketMQ-维基百科](https://zh.wikipedia.org/wiki/Apache_RocketMQ)

[RocketMQ-8.2.0官方文档](http://rocketmq.apache.org/docs/quick-start/)

[RocketMQ扩展项目集合](https://github.com/apache/rocketmq-externals)

[[docker部署rocketmq深度历险！各种bug带你解决](https://www.cnblogs.com/pc-m/p/11046848.html)](https://www.cnblogs.com/pc-m/p/11046848.html)

[linux脚本相关： Syntax error: end of file unexpected (expecting "then") 提示错误](https://www.cnblogs.com/langlang/archive/2010/11/27/1889664.html)

[报错-bash: ./a.sh: Permission denied](https://blog.csdn.net/Mr_xiao_1/article/details/83651367)