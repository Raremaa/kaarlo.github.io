---
title: "Kafka基础"
date: 2021-02-23 10:18:29.198
draft: false
type: "post"
showTableOfContents: true
tags: ["Kafka"]
---

# Kafka基础

# 1. Kafka基础概念

## 1.1. Kafka是什么

Kafka是一个**拥有流处理功能的消息引擎系统**。

主要有五大部分：

- **Producer API**，主要用来发送消息。
- **Consumer API**，主要用来消费消息。
- **Streams API**，Kafka流处理API，这是一个仅适用Kafka的轻量级流处理客户端库。
- **Connect API**，Kafka连接器API，串联Kafka上下游系统

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled.png](https://img.masaiqi.com/20210223101726.png)

- **Admin API**， Kafka管理API，可以用来管理Kafka的Topics，Brokers等Kafka对象。

## 1.2. Kafka的版本号

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%201.png](https://img.masaiqi.com/20210223101727.png)

简单来说，kafka_用来编译的Scala版本_kafka版本

## 1.3. Kafka的部分术语

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%202.png](https://img.masaiqi.com/20210223101728.png)

- **消息：Record**。Kafka 是消息引擎嘛，这里的消息就是指 Kafka 处理的主要对象。
- **主题：Topic**。主题是承载消息的逻辑容器，在实际使用中多用来区分具体的业务。
- **分区：Partition**。一个有序不变的消息序列。每个主题下可以有多个分区。**分区是主题下的概念，每个主题划分成多个分区**，供消费组消费。
- **消息位移：Offset**。表示分区中每条消息的位置信息，是一个单调递增且不变的值。**LEO(Log End offset)表征副本当前下一条要写入消息的位置**(比如当前写入到位移0，位移0有数据，则LEO为1)
- **副本：Replica**。Kafka 中同一条消息能够被拷贝到多个地方以提供数据冗余，这些地方就是所谓的副本。副本还分为领导者副本和追随者副本，各自有不同的角色划分。副本是在分区层级下的，即每个分区可配置多个副本实现高可用。
- **Kafka服务端：Broker**。
- **生产者：Producer**。向主题发布新消息的应用程序。
- **消费者：Consumer**。从主题订阅新消息的应用程序。
- **消费者位移：Consumer Offset**。表征消费者消费进度，每个消费者都有自己的消费者位移。**Kafka消费位移是记录下一条要消费的消息的位移。**
- **消费者组：Consumer Group**。多个消费者实例共同组成的一个组，同时消费多个分区以实现高吞吐。
- **重平衡：Rebalance。**消费者组内某个消费者实例挂掉后，其他消费者实例自动重新分配订阅主题分区的过程。Rebalance 是 Kafka 消费者端实现高可用的重要手段。

这里有几个需要注意的点：

1. **多个Broker构成Kafka集群，一个Broker上会有很多分区(可以来自不同主题)。**
2. **主题与分区一对多**。分区是定义在主题下的概念，**一个主题下多个分区，可以分散在不同Broker上**
3. **消费者组与主题多对多**，**一个消费组可以消费多个主题**，一个主题可以被多个消费组消费(消费位移(进度)数据彼此独立)
4. **每个分区只能由同一个消费者组内的一个 Consumer 实例来消费**，多个消费组每个消费组只能有一个Consumer实例消费这个分区。

## 1.4. Kafka消息日志

**Kafka 使用消息日志（Log）来保存数据**，一个日志就是磁盘上一个只能**追加写（Append-only）**消息的物理文件。因为**只能追加写入，故避免了缓慢的随机 I/O 操作，改为性能较好的顺序 I/O 写操作，这也是实现 Kafka 高吞吐量特性的一个重要手段**。

因此**一般认为用机械硬盘部署Kafka就好，SSD对Kafka提升不大，因为SSD的优点在于随机读写性能优异，对于Kafka的追加写并明显提升**。

**Kafka 通过日志段（Log Segment）机制进行日志删除**。在 Kafka 底层，一个日志又进一步细分成多个日志段，消息被追加写到当前最新的日志段中，当写满了一个日志段后，Kafka 会自动切分出一个新的日志段，并将老的日志段封存起来。Kafka 在后台还有定时任务会定期地检查老的日志段是否能够被删除，从而实现回收磁盘空间的目的。

# 2. Kafka的分区策略

Kafka为了增加系统的伸缩性(Scalability)，引入了分区(Partitioning)的概念。

Kafka 中的分区机制指的是将每个主题划分成多个分区（Partition），每个分区是一组有序的消息日志。**主题下的每条消息只会保存在某一个分区中，而不会在多个分区中被保存多份。**

通过这个设计，就可以以分区这个粒度进行数据读写操作，每个Broker的各个分区独立处理请求，进而实现负载均衡，提升了整体系统的吞吐量。

**分区策略是决定生产者将消息发送到哪个分区的算法。**

- **轮询策略**，即按消息顺序进行分区顺序分配(比如图中消息顺序1，2，3，4...会按顺序分配在各个分区中)

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%203.png](https://img.masaiqi.com/20210223101729.png)

一般来说，有3种类型的分区策略

- **随机策略**。这是老版本Kafka的默认策略。
- **Key-ordering策略**。有点类似哈希桶算法，对于有key的数据，用key的哈希值对分区数取模计算对应的分区。

    ```java
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    return Math.abs(key.hashCode()) % partitions.size();
    ```

    Kafka默认：

    - 如果指定了 Key，那么默认实现按Key-ordering策略；
    - 如果没有指定 Key，则使用轮询策略。

    如果要自定义分区策略，可以显式地配置生产者端的参数`partitioner.class`。即实现`org.apache.kafka.clients.producer.Partitioner`接口。这个接口只定义了两个方法：`#partition()`和`#close()`。

# 3. Kafka的生产者压缩算法

一般遵循**Producer 端压缩、Broker 端保持、Consumer 端解压缩**。
生产者Producer需要按照配置压缩消息成指定格式发送给Broker端，Broker端需要解压缩并进行消息校验，如果需要转化成其他压缩格式则需要再压缩(浪费CPU资源)，否则发送之前保存的已经压缩的消息内容，交由Consumer端来解压缩。

下图展示了常用压缩算法的压缩比，压缩和解压缩速度：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%204.png](https://img.masaiqi.com/20210223101730.png)

# 4. Kafka消息丢失相关

- **Producer发送消息到Broker上丢失消息。**中间可能因为网络抖动等原因，消息并没有真正发送到Broker上。

    因此，**一般建议永远使用带回调(异步有异步回调)的发送方法**。

- **消费者程序记录错位移信息丢失消息。**

    消费者消费消息有一个消费位移，要**维持先消费消息，再更新位移的顺序**。

    除此以外，异步线程处理时可能会失败，但如果自动提交位移消息会提交位移，但是部分消息没有被消费成功，因此**建议多线程环境手动提交消息，不要使用自动提交位移**。

    当然上述方案都可能带来消息重复消费的问题。

胡夕老师总结了一些**最佳实践**经验：

1. 不要使用 `producer.send(msg)`，而要使用 `producer.send(msg, callback)`。记住，一定要使用带有回调通知的 send 方法。
2. 设置 `acks = all`。`acks` 是 Producer 的一个参数，代表了你对“已提交”消息的定义。如果设置成 `all`，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。
3. 设置 `retries` 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 `retries` > 0 的 Producer 能够自动重试消息发送，避免消息丢失。
4. 设置 `unclean.leader.election.enable = false`。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，即不允许这种情况的发生。
5. 设置 `replication.factor >= 3`。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。
6. 设置 `min.insync.replicas > 1`。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。
7. 确保 `replication.factor > min.insync.replicas`。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 `replication.factor = min.insync.replicas + 1`。
8. 确保消息消费完成再提交。Consumer 端有个参数 `enable.auto.commit`，最好把它设置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程处理的场景而言是至关重要的。

# 5. Kafka基于TCP进行通信

**Apache Kafka 的所有通信都是基于 TCP** 的，无论是生产者、消费者，还是 Broker 之间的通信都是如此。

## 5.1. 生产者端的TCP管理

- 何时建立TCP连接
    1. **在创建 KafkaProducer 实例时，生产者应用会在后台创建并启动一个名为 Sender 的线程，该 Sender 线程开始运行时首先会创建与 Broker 的连接(即使没有调用`#send`方法)**。

        **具体来说，是Producer端的`bootstrap.servers`参数，指定了需要在启动时建立TCP连接的Broker地址，一般推荐设置3～5个为宜，不用全部设置**，事实上只要和其中一个Broker建立了TCP连接，当前Producer是能够获取到集群所有Broker的信息的。

    2. **消息发送时，如果发现与某些Broker没有建立TCP连接，会建立TCP连接**。
    3. **更新集群元数据信息时也会建立TCP连接**(比如Topic不存在时会发送METADATA请求给Kafka集群获取最新元数据信息)
- 何时关闭TCP连接
    1. **用户主动关闭**。(比如kill -9 或者调用close方法)
    2. **Kafka自动关闭**，对应Producer 端参数 `connections.max.idle.ms`。默认情况下该参数值是 9 分钟，即如果在 9 分钟内没有任何请求“流过”某个 TCP 连接，那么 Kafka 会主动帮你把该 TCP 连接关闭。**注意，这个关闭是Broker端发起，Producer属于passive close(被动关闭)**，被动关闭的后果就是会产生大量的 CLOSE_WAIT 连接，因此 **Producer 端或 Client 端没有机会显式地观测到此连接已被中断**。

## 5.2. 消费者端的TCP管理

- 何时建立TCP连接
    1. **在调用 `KafkaConsumer.poll` 方法时被创建TCP连接**。

        具体来说是围绕协调者进行的交互：

        - **发起 FindCoordinator 请求时**。向Broker端发起 FindCoordinator 请求(建立TCP连接)，寻找协调者所在Broker。
        - **连接协调者时**。确定协调者所在Broker，建立TCP连接
        - **消费数据时。**协调者分配好当前消费者消费的分区后，**消费者需要与这些分区领导者副本所在的Broker建立TCP连接**。
- 何时关闭TCP连接
    1. **用户主动关闭**。(比如kill -9 或者调用close方法)
    2. **Kafka自动关闭**，对应消费者端参数 `connection.max.idle.ms`。默认情况下该参数值是 9 分钟，即如果在 9 分钟内没有任何请求“流过”某个 TCP 连接，消费者会强行“杀掉”这个 Socket 连接。

# 6. Kafka消息交付可靠性保障

所谓的消息交付可靠性保障，是指 Kafka 对 Producer 和 Consumer 要处理的消息提供什么样的承诺。常见的承诺有以下三种：

- **最多一次（at most once）**：消息可能会丢失，但绝不会被重复发送。
- **至少一次（at least once）**：消息不会丢失，但有可能被重复发送。
- **精确一次（exactly once）**：消息不会丢失，也不会被重复发送。

目前，**Kafka 默认提供的交付可靠性保障是第二种，即至少一次**。

假设消息成功“提交”，但 Broker 的应答没有成功发送回 Producer 端（比如**网络出现瞬时抖动**），那么 **Producer 就无法确定消息是否真的提交成功了。因此，它只能选择重试，也就是再次发送相同的消息，回带来消息重复发送问题。**

**实现最多一次只需要禁止重试机制就可以实现。**

**Kafka实现精准一次语义需要基于幂等性（Idempotence）和事务（Transaction）**。

## 6.1. Kafka消息幂等性

设置Producer端参数props.put(“enable.idempotence”, ture)，或props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG， true)就可以开启Kafka幂等消息。

Producer 升级成幂等性 Producer后，其他所有的代码逻辑都不需要改变。Kafka 自动帮你做消息的重复去重。

**它只能保证单分区上的幂等性，无法保证跨分区，跨会话的幂等性**。即一个幂等性 Producer 能够保证某个主题的一个分区上不出现重复消息，它无法实现多个分区的幂等性。其次，它只能实现单会话上的幂等性，不能实现跨会话的幂等性。这里的会话，你可以理解为 Producer 进程的一次运行。当你重启了 Producer 进程之后，这种幂等性保证就丧失了。

## 6.2. Kafka消息事务性

设置事务型 Producer ，需要满足两个要求：

- 和幂等性 Producer 一样，开启 `enable.idempotence = true`。
- 设置 Producer 端参数 transactional. id。最好为其设置一个有意义的名字。

此外，Producer端代码也需要一些事务性的代码:

```java
producer.initTransactions();
try {
            producer.beginTransaction();
            producer.send(record1);
            producer.send(record2);
            producer.commitTransaction();
} catch (KafkaException e) {
            producer.abortTransaction();
}
```

**实际上即使写入失败，Kafka 也会把它们写入到底层的日志中**，也就是说 Consumer 还是会看到这些消息。因此在 Consumer 端，读取事务型 Producer 发送的消息也是需要一些变更的，设置 `isolation.level` 参数的值即可:

- **read_uncommitted**：这是默认值，表明 Consumer 能够读取到 Kafka 写入的任何消息，不论事务型 Producer 提交事务还是终止事务，其写入的消息都可以读取。**如果你用了事务型 Producer，那么对应的 Consumer 就不要使用这个值。**
- **read_committed**：表明 Consumer 只会读取事务型 Producer 成功提交事务写入的消息。当然了，它也能看到非事务型 Producer 写入的所有消息。

Kafka官网，对事务有如下论述：

Transactional delivery allows producers to send data to multiple partitions such that either all messages are successfully delivered, or none of them are.

大概是说，**Kafka事务能够保证多分区消息发布时要么全部成功，要么全部失败。(原子性)**

# 7. Kafka消费者组

## 7.1. Kafka消费组特性

Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制。它们**共享一个公共的 ID，这个 ID 被称为 Group ID**。组内的所有消费者协调在一起来消费**单个或多个订阅主题**（Subscribed Topics）的所有分区（Partition）。**每个分区只能由同一个消费者组内的一个 Consumer 实例来消费**。(不同消费组会有多个Consumer来消费，也就是说**一个分区可以被多个Consumer Group进行消费**)

- 一般的消息队列模型
    1. **传统点对点模型**是消息一旦被消费，就会从队列中被删除，而且只能被下游的一个 Consumer 消费。这既是特性，也是一种伸缩性比较差的缺陷。
    2. **传统发布/订阅模型**允许消息被多个 Consumer 消费，但它的问题也是伸缩性不高，因为每个订阅者都必须要订阅消费主题的所有分区。
- Kafka的消费者组解决的问题

    **Kafka使用 Consumer Group 这一种机制，可以实现了传统消息引擎系统的两大模型。除此以外，因为同个消费组多个消费者实例共同分担消费各个Topic分区，也就提高了伸缩性。**

**理想情况下，Consumer 实例的数量应该等于该 Group 订阅主题的分区总数，**这样每个Consumer实例理想情况都会分担一个分区都消费任务，也不会造成资源浪费(牺牲可用性)。

## 7.2. Kafka消费位移

针对 Consumer Group，Kafka需要记录当前分区当前消费组消费的位置，这个位置我们称之为位移（Offset）。**Kafka消费位移是记录下一条要消费的消息的位移**，而不是目前最新消费消息的位移。

**Consumer 需要向 Kafka 汇报自己的位移数据，这个汇报过程被称为提交位移（Committing Offsets）。**

因为 Consumer 能够同时消费多个分区的数据，所以位移的提交实际上是在分区粒度上进行的，即 **Consumer 需要为分配给它的每个分区提交各自的位移数据**。

如果提交了位移x，那么认为所有小于位移x的消息都已经被消费。**位移提交的语义保障是由消费者来负责的，Kafka 只会“无脑”地接受消费者提交的位移**。对位移提交的管理直接影响了Consumer 所能提供的消息语义保障。

**Kafka Consumer 提交位移的方式有两种：自动提交位移和手动提交位移(包括同步提交与异步提交)**。

- **自动提交位移**

    Consumer 端有个参数叫 `enable.auto.commit`，如果值是 true，则 Consumer 在后台默默地为你定期提交位移，提交间隔由一个专属的参数 `auto.commit.interval.ms` 来控制。

    **一旦设置了 `enable.auto.commit` 为 true，Kafka 会保证在开始调用 poll 方法时，提交上次 poll 返回的所有消息。**从顺序上来说，poll 方法的逻辑是先提交上一批消息的位移，再处理下一批消息，因此它能保证不出现消费丢失的情况。但**自动提交位移的一个问题在于，它可能会出现重复消费**。

    在默认情况下，Consumer 每 5 秒自动提交一次位移。现在，我们假设提交位移之后的 3 秒发生了 Rebalance 操作。在 Rebalance 之后，所有 Consumer 从上一次提交的位移处继续消费，但该位移已经是 3 秒前的位移数据了，故在 Rebalance 发生前 3 秒消费的所有数据都要重新再消费一次。虽然你能够通过减少 `auto.commit.interval.ms` 的值来提高提交频率，但这么做只能缩小重复消费的时间窗口，不可能完全消除它。这是自动提交机制的一个缺陷。

    自动提交位移很省事，但是不够灵活，而且由于是自动提交位移，**即使当前位移主题(下文表述)没有消息可以消费了，位移主题中还是会不停地写入最新位移的消息**。显然 Kafka 只需要保留这类消息中的最新一条就可以了，之前的消息都是可以删除的。这就要求 **Kafka 必须要有针对位移主题消息特点的消息删除策略**，否则这种消息会越来越多，最终撑爆整个磁盘。**Kafka 使用 Compact 策略来删除位移主题中的过期消息。**Kafka 提供了**专门的后台线程Log Cleaner定期地巡检待 Compact 的主题，看看是否存在满足条件的可删除数据**。

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%205.png](https://img.masaiqi.com/20210223101731.png)

- **手动提交位移**

    事实上，很多与 Kafka 集成的大数据框架都是禁用自动提交位移的，如 Spark、Flink 等。

    **手动提交位移，即设置 `enable.auto.commit = false`**。一旦设置了 false，作为 Consumer 应用开发的你就要承担起位移提交的责任。**处理完了 poll() 方法返回的所有消息之后再提交位移，否则有可能出现消息丢失(消息没有处理完或出现异常)**

    Kafka Consumer API主要提供了两种手动提交位移的方式：

    1.  `**consumer.commitSync` 同步提交，阻塞式，会自动重试**。

        关于这个自动重试，可以看一下Java API源码，本质上是基于超时时间的一个do-while循环：

        ```java
        //...
        public void commitSync(Duration timeout) {
                acquireAndEnsureOpen();
                try {
                    maybeThrowInvalidGroupIdException();
                    if (!coordinator.commitOffsetsSync(subscriptions.allConsumed(), time.timer(timeout))) {
                        throw new TimeoutException("Timeout of " + timeout.toMillis() + "ms expired before successfully " +
                                "committing the current consumed offsets");
                    }
                } finally {
                    release();
                }
        }
        //...

        public boolean commitOffsetsSync(Map<TopicPartition, OffsetAndMetadata> offsets, Timer timer) {
                invokeCompletedOffsetCommitCallbacks();

                if (offsets.isEmpty())
                    return true;

                do {
                    if (coordinatorUnknown() && !ensureCoordinatorReady(timer)) {
                        return false;
                    }

                    RequestFuture<Void> future = sendOffsetCommitRequest(offsets);
                    client.poll(future, timer);

                    // We may have had in-flight offset commits when the synchronous commit began. If so, ensure that
                    // the corresponding callbacks are invoked prior to returning in order to preserve the order that
                    // the offset commits were applied.
                    invokeCompletedOffsetCommitCallbacks();

                    if (future.succeeded()) {
                        if (interceptors != null)
                            interceptors.onCommit(offsets);
                        return true;
                    }

                    if (future.failed() && !future.isRetriable())
                        throw future.exception();

                    timer.sleep(rebalanceConfig.retryBackoffMs);
                } while (timer.notExpired());

                return false;
            }
        ```

    2. `**consumer.commitSync`  异步提交，不阻塞，无法重试**(异步提交，位移可能过期，重试没有意义)。

    **建议将同步与非同步提交位移结合使用，既可以拥有异步提交的非阻塞特性，又可以拥有同步阻塞+自动重试的保障**：

    ```java
    try {
               while(true) {
                            ConsumerRecords<String, String> records = 
                                        consumer.poll(Duration.ofSeconds(1));
                            process(records); // 处理消息
                            commitAysnc(); // 使用异步提交规避阻塞
                }
    } catch(Exception e) {
                handle(e); // 处理异常
    } finally {
                try {
                            consumer.commitSync(); // 最后一次提交使用同步阻塞式提交
    					  } finally {
    									      consumer.close();
    						}
    }
    ```

    有两个比较特殊的提交方式：`**commitSync(Map<TopicPartition, OffsetAndMetadata>)` 和 `commitAsync(Map<TopicPartition, OffsetAndMetadata>`)，这两个API可以实现批量提交一组位移信息**，而不是一个一个提交。以一个具体例子看，下面的代码实现了每100条消息提交一次：

    ```java
    private Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
    int count = 0;
    ……
    while (true) {
                ConsumerRecords<String, String> records = 
      consumer.poll(Duration.ofSeconds(1));
                for (ConsumerRecord<String, String> record: records) {
                            process(record);  // 处理消息
                            offsets.put(new TopicPartition(record.topic(), record.partition()),
                                       new OffsetAndMetadata(record.offset() + 1)；
                           if（count % 100 == 0）
                                        consumer.commitAsync(offsets, null); // 回调处理逻辑是null
                            count++;
      }
    }
    ```

## 7.3. Kafka位移主题

老的Kafka版本是将位移信息存储在Apache ZooKeeper这个分布式的协调服务框架中，但**ZooKeeper 这类元框架其实并不适合进行频繁的写更新，而 Consumer Group 的位移更新却是一个非常频繁的操作。这种大吞吐量的写操作会极大地拖慢 ZooKeeper 集群的性能**。

新版本的Kafka中， **Consumer Group 将位移保存在 Broker 端的内部主题**中，也就是**位移主题**。

直观感受一下，看一下Kafka日志的截图：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%206.png](https://img.masaiqi.com/20210223101732.png)

这些`__consumer_offsets-xxx`就是Kafka的位移主题，Kafka将 Consumer 的位移数据作为一条条普通的 Kafka 消息，提交到 __consumer_offsets 中。

**位移主题是一个普通的 Kafka 主题，但它的消息格式却是 Kafka 自己定义的**，用户不能修改，也就是说你**不能随意地向这个主题写消息，因为一旦你写入的消息不满足 Kafka 规定的格式，那么 Kafka 内部无法成功解析，就会造成 Broker 的崩溃。**

这里的消息格式可以简单理解为一组KV对：

- 位移主题的 Key 中应该保存 3 部分内容：<Group ID，主题名，分区号 >。
- 位移主题的Value也就是消息体有三种：
    1. 位移信息与一些元数据信息(比如时间戳等)
    2. 用于保存 Consumer Group 信息的消息。
    3. 用于删除 Group 过期位移甚至是删除 Group 的消息。它有个专属的名字：tombstone 消息，即墓碑消息，也称 delete mark。一旦某个 Consumer Group 下的所有 Consumer 实例都停止了，而且它们的位移数据都已被删除时，Kafka 会向位移主题的对应分区写入 tombstone 消息，表明要彻底删除这个 Group 的信息。
- 位移主题何时被创建

    **当 Kafka 集群中的第一个 Consumer 程序启动时，Kafka 会自动创建位移主题**。

    位移主题分区数可以通过Broker 端参数 `offsets.topic.num.partitions` 的取值了。它的默认值是 50，因此 Kafka 会默认自动创建一个 50 分区的位移主题。

    副本数或备份因子可以通过`offsets.topic.replication.factor`进行设置，它的默认值为3，因此Kafka会默认创建3个副本。

## 7.4. Kafka重平衡（Rebalance）

### 7.4.1. 重平衡概述

**Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 Consumer 如何达成一致，来分配订阅 Topic 的每个分区。简单来说，就是重新分配分区。**

- 何时触发Rebalance
    1. **组成员数发生变更**。比如有新的 Consumer 实例加入组或者离开组，抑或是有 Consumer 实例崩溃被“踢出”组。
    2. **订阅主题数发生变更**。Consumer Group 可以使用正则表达式的方式订阅主题，比如 consumer.subscribe(Pattern.compile("t.*c")) 就表明该 Group 订阅所有以字母 t 开头、字母 c 结尾的主题。在 Consumer Group 的运行过程中，你新创建了一个满足这样条件的主题，那么该 Group 就会发生 Rebalance。
    3. **订阅主题的分区数发生变更**。Kafka 当前只能允许增加一个主题的分区数。当分区数增加时，就会触发订阅该主题的所有 Group 开启 Rebalance。
- Rebalance过程的痛点

    Rebalance 过程对 Consumer Group 消费过程有极大的影响。如果你了解 JVM 的垃圾回收机制，你一定听过万物静止的收集方式，即著名的 stop the world，简称 STW。在 STW 期间，所有应用线程都会停止工作，表现为整个应用程序僵在那边一动不动。**Rebalance 过程也和这个类似，在 Rebalance 过程中，所有 Consumer 实例都会停止消费，等待 Rebalance 完成**。这是 Rebalance 为人诟病的一个方面。

### 7.4.2. Kafka Coordinator组件

**负责为 Group 执行 Rebalance 以及提供位移管理和组成员管理的“协调者”，在Kafka中称为Coordinator。**

**所有 Broker 在启动时，都会创建和开启相应的 Coordinator 组件。也就是说，所有 Broker 都有各自的 Coordinator 组件。**

**Consumer 端应用程序在提交位移时，其实是向 Coordinator 所在的 Broker 提交位移**。

- **Kafka如何为Consumer Group 确定 Coordinator 所在的 Broker，分两步**：
    1. **确定由位移主题的哪个分区来保存该 Group 数据：**

        ```java
        partitionId = Math.abs(groupId.hashCode() % offsetsTopicPartitionCount)
        ```

    2. **找出该分区 Leader 副本所在的 Broker，该 Broker 即为对应的 Coordinator**

### 7.4.3. 减少重平衡的方案

Rebalance对系统性能有较大影响，需要区分哪些Rebalance是必要的，不可避免；哪些Rebalance是不必要的，可以避免，进而提升系统性能。

- **不可避免的Rebalance(常规操作)**

    上文触发时机后两点：**订阅主题数发生变更 + 订阅主题的分区数发生变更，是运维主动操作，不可避免**

    对于**组成员数发生变更，增多往往是注册了新的Consumer实例进Consumer Group；减少往往是主动停掉Consumer实例，都是常规操作，一般不可避免。**

- **可避免的Rebalance(意外情况)**

    **意外情况导致Consumer 实例会被 Coordinator 错误地认为“已停止”从而被“踢出”Group。**

    Consumer 端有3个参数：

    1. `session.timeout.ms`：Coordinator通过与Consumer的心跳检测是否不可用，这个值控制超时时间。
    2. `heartbeat.interval.ms`：Coordinator与Consumer心跳的间隔时间。
    3. `max.poll.interval.ms`：Consumer消费消息的最大时间，超时则认为不可用。

    首先，**要避免未能及时发送心跳，导致 Consumer 被“踢出”Group 而引发的Rebalance**，胡夕老师推荐配置如下：

    1. 设置 `session.timeout.ms` = 6s。
    2. 设置 `heartbeat.interval.ms` = 2s。
    3. 要保证 Consumer 实例在被判定为“dead”之前，能够发送至少 3 轮的心跳请求，即 `session.timeout.ms` >= 3 * `heartbeat.interval.ms`。

    其次，**要避免Consumer 消费时间过长导致的Rebalance**，也就是设置`max.poll.interval.ms`的值，这个要根据业务处理时间去设置。

### 7.4.4. 重平衡过程如何通知到所有Consumer实例

**重平衡过程是靠消费者端的心跳线程（Heartbeat Thread）通知其他消费者实例的**。当协调者决定开启新一轮重平衡后，它会将`REBALANCE_IN_PROGRESS`封装进心跳请求的响应中，发还给消费者实例。当消费者实例发现心跳响应中包含了`REBALANCE_IN_PROGRESS`，就能立马知道重平衡又开始了，这就是重平衡的通知机制。当然**这个就对应上文的`heartbeat.interval.ms`参数，可以控制心跳发送间隔时间。**

### 7.4.5. 消费组状态机

**Kafka 设计了一套消费者组状态机（State Machine），来帮助协调者完成整个重平衡流程**，目前定义了五种状态：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%207.png](https://img.masaiqi.com/20210223101733.png)

状态流转：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%208.png](https://img.masaiqi.com/20210223101734.png)

一个消费者组最开始是 `Empty` 状态，当重平衡过程开启后，它会被置于 `PreparingRebalance` 状态等待成员加入，之后变更到 `CompletingRebalance` 状态等待分配方案，最后流转到 `Stable` 状态完成重平衡。

当有新成员加入或已有成员退出时，消费者组的状态从 `Stable` 直接跳到 `PreparingRebalance` 状态，此时，所有现存成员就必须重新申请加入组。当所有成员都退出组后，消费者组状态变更为 `Empty`。Kafka 定期自动删除过期位移的条件就是，组要处于 `Empty` 状态。因此，如果你的消费者组停掉了很长时间（超过 7 天），那么 Kafka 很可能就把该组的位移数据删除了，日志表现如下：

```bash
Removed ✘✘✘ expired offsets in ✘✘✘ milliseconds.
```

**只有 Empty 状态下的组，才会执行过期位移删除的操作**。

### 7.4.6. 重平衡流程

*从消费者角度看，在消费者端，重平衡分为两个步骤*：

1. 加入组，对应`JoinGroup`请求。

    当组内成员加入组时，它会向协调者发送 `JoinGroup` 请求。在该请求中，每个成员都要将自己订阅的主题上报，这样协调者就能收集到所有成员的订阅信息。一旦收集了全部成员的 JoinGroup 请求后，协调者会从这些成员中选择一个担任这个消费者组的领导者。通常情况下，第一个发送 JoinGroup 请求的成员自动成为“领导者”。**这里的领导者是一个Consumer实例，职责是收集所有成员的订阅信息，然后根据这些信息，制定具体的分区消费分配方案。**

    选出领导者之后，协调者会把消费者组订阅信息封装进 JoinGroup 请求的响应体中，然后发给领导者，由领导者统一做出分配方案后，进入到下一步：发送 `SyncGroup` 请求。

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%209.png](https://img.masaiqi.com/20210223101735.png)

2. 等待领导者消费者（Leader Consumer）分配方案，对应`SyncGroup`请求。

    在这一步中，领导者向协调者发送 SyncGroup 请求，将刚刚做出的分配方案发给协调者。值得注意的是，其他成员也会向协调者发送 SyncGroup 请求，只不过请求体中并没有实际的内容。这一步的主要目的是让协调者接收分配方案，然后统一以 SyncGroup 响应的方式分发给所有成员，这样组内所有成员就都知道自己该消费哪些分区了。

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2010.png](https://img.masaiqi.com/20210223101736.png)

当所有成员都成功接收到分配方案后，消费者组进入到 Stable 状态，即开始正常的消费工作。

*从Broker角度看，分为几个场景*：

1. 新成员入组

    新成员入组是指组处于 Stable 状态后，有新成员加入。

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2011.png](https://img.masaiqi.com/20210223101737.png)

2. 组成员主动离组

    消费者实例所在线程或进程调用 close() 方法主动通知协调者它要退出。这个场景就涉及到了第三类请求：`LeaveGroup` 请求。协调者收到 `LeaveGroup` 请求后，依然会以心跳响应的方式通知其他成员。

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2012.png](https://img.masaiqi.com/20210223101738.png)

3. 组成员崩溃离组

    崩溃离组是指消费者实例出现严重故障，突然宕机导致的离组。它和主动离组是有区别的，因为后者是主动发起的离组，协调者能马上感知并处理。但崩溃离组是被动的，协调者通常需要等待一段时间才能感知到，这段时间一般是由消费者端参数 `session.timeout.ms` 控制的。

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2013.png](https://img.masaiqi.com/20210223101739.png)

4. 重平衡时协调者对组内成员提交位移的处理

    正常情况下，每个组内成员都会定期汇报位移给协调者。当重平衡开启时，协调者会给予成员一段缓冲时间，要求每个成员必须在这段时间内快速地上报自己的位移信息，然后再开启正常的 `JoinGroup/SyncGroup` 请求发送。

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2014.png](https://img.masaiqi.com/20210223101740.png)

## 7.5. Kafka Consumer多线程方案

**Kafka Consumer 包含用户主线程和心跳线程**。所谓用户主线程，就是你启动 Consumer 应用程序 main 方法的那个线程，而新引入的心跳线程（Heartbeat Thread）只负责定期给对应的 Broker 机器发送心跳请求，以标识消费者应用的存活性（liveness）。

很明显，**Kafka Consumer非线程安全**，举个例子，就提交位移来说，如果多线程A和B共享一个Consumer实例，A线程还没处理完，B线程提交了，这里就有丢失消息的风险。

我们的操作基本都是基于用户主线程，从这个纬度可以将Kafka Consumer当做单线程的设计。那么，我们有两套实现多线程Consumer的方案：

1. **消费者程序启动多个线程，每个线程维护专属的 KafkaConsumer 实例，负责完整的消息获取、消息处理流程。**

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2015.png](https://img.masaiqi.com/20210223101741.png)

2. **消费者程序使用单或多线程获取消息，同时创建多个消费线程执行消息处理逻辑。获取消息的线程可以是一个，也可以是多个，每个线程维护专属的 KafkaConsumer 实例，处理消息则交由特定的线程池来做，从而实现消息获取与消息处理的真正解耦。**

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2016.png](https://img.masaiqi.com/20210223101742.png)

两个方案比较如下：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2017.png](https://img.masaiqi.com/20210223101743.png)

## 7.6. Kafka消费进度监控

1. 使用 Kafka 自带的命令行工具 kafka-consumer-groups 脚本。
2. 使用 Kafka Java Consumer API 编程。
3. 使用 Kafka 自带的 JMX 监控指标(可以使用JConsole工具查看)。

# 8. Kafka副本机制(补充领导者副本选举过程)

## 8.1. Kafka副本机制概述

**副本机制（Replication），也可以称之为备份机制，通常是指分布式系统在多台网络互联的机器上保存有相同的数据拷贝。**

副本机制的优点

- **提供数据冗余(Kafka可以体现)**。即使系统部分组件失效，系统依然能够继续运转，因而增加了整体可用性以及数据持久性。
- **提供高伸缩性**。支持横向扩展，能够通过增加机器的方式来提升读性能，进而提高读操作吞吐量。
- **改善数据局部性**。允许将数据放入与用户地理位置相近的地方，从而降低系统延时。

Kafka的的副本仅可以体现数据冗余的好处。

副本（Replica），本质就是一个只能追加写消息的提交日志。

**Kafka的副本是在分区下定义的概念，同一个分区下的所有副本保存有相同的消息序列，这些副本分散保存在不同的 Broker 上**，从而能够对抗部分 Broker 宕机带来的数据不可用。

**Kafka将副本分成两类：领导者副本（Leader Replica）和追随者副本（Follower Replica）。每个分区在创建时都要选举一个副本，称为领导者副本，其余的副本自动称为追随者副本**：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2018.png](https://img.masaiqi.com/20210223101744.png)

领导者副本接受生产者的数据读写请求，追随者副本**异步拉取**领导者副本的消息写入到自己提交日志中进而与领导者副本同步。**注意，所有的读写请求都必须发往领导者副本所在的 Broker，由该 Broker 负责处理，追随者副本不对外提供任何服务**（不像Mysql可以抗读请求）。

这样的设计有如下好处：

1. **Read-your-writes，即立刻读到你写的内容**。读写都在一个领导者副本所在Broker上，只要写入一定能立刻读到。如果允许追随者副本提供服务，那么有可能在数据拉取期间读请求打到追随者副本上，进而没法读到刚写的数据。
2. **Monotonic Reads(单调读)，即不会出现某条消息一会儿“读到”，一会“读不到”**。如果允许追随者副本提供服务，那么不一样到追随者副本根据同步进度不一样不一定数据一致。

## 8.2. In-sync Replicas（ISR）

**Kafka 引入了 In-sync Replicas，也就是所谓的 ISR 集合用来描述一个追随者副本是否与Leader 同步** 。ISR 中的副本都是与 Leader 同步的副本，相反，不在 ISR 中的追随者副本就被认为是与 Leader 不同步的。

**注意，Leader副本天然就在ISR集合中。**

Broker 端有个参数 `replica.lag.time.max.ms` 。这个参数的含义是 Follower 副本能够落后 Leader 副本的最长时间间隔，当前默认值是 10 秒。**注意这里的时间间隔是指同步过程的速度持续慢于 Leader 副本的消息写入速度的值**。只要一个 Follower 副本落后 Leader 副本的时间不连续超过 10 秒，那么 Kafka 就认为该 Follower 副本与 Leader 是同步的。

如果超时则会被“踢出”ISR集合，如果后面Follower副本追上了Leader进度，也会再加入ISR中，**ISR 是一个动态调整的集合，而非静态不变的。**

## 8.3. Unclean 领导者选举（Unclean Leader Election）

一般的Leader选举都是在ISR集合中进行选举，**Kafka 把所有不在 ISR 中的存活副本都称为非同步副本，允许这些副本参与Leader选举称为Unclean 领导者选举**。

Broker 端参数 `unclean.leader.election.enable` 控制是否允许 Unclean 领导者选举。(建议不开启)

这其实是经典CAP 理论的博弈，**一个分布式系统通常只能同时满足一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）中的两个。显然，在这个问题上，Kafka 赋予你选择 C 或 A 的权利。**

# 9. Kafka Broker如何处理请求

**Kafka使用Reactor 模式处理客户端请求。Reactor 模式是事件驱动架构的一种实现方式**，特别适合应用于处理多个客户端并发向服务器端发送请求的场景：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2019.png](https://img.masaiqi.com/20210223101745.png)

多个客户端会发送请求给到 Reactor。Reactor 有个请求分发线程 Dispatcher，也就是图中的 Acceptor，它会将不同的请求下发到多个工作线程中处理。

在这个架构中，Acceptor 线程只是用于请求分发，不涉及具体的逻辑处理，非常得轻量级，因此有很高的吞吐量表现。而这些工作线程可以根据实际业务处理需要任意增减，从而动态调节系统负载能力。

Kafka中的流程如下：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2020.png](https://img.masaiqi.com/20210223101746.png)

Kafka 的 Broker 端有个 SocketServer 组件，类似于 Reactor 模式中的 Dispatcher，它也有对应的 Acceptor 线程和一个工作线程池，只不过在 Kafka 中，这个**工作线程池有个专属的名字，叫网络线程池**。Kafka 提供了 Broker 端参数 `num.network.threads`，用于调整该网络线程池的线程数。其默认值是 3，表示每台 Broker 启动时会创建 3 个网络线程，专门处理客户端发送的请求。

网络线程接收后，Kafka 在这个环节又做了一层异步线程池的处理：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2021.png](https://img.masaiqi.com/20210223101747.png)

当**网络线程拿到请求后将请求放入到一个共享请求队列中**。**Broker 端还有个 IO 线程池，负责从该队列中取出请求，执行真正的处理**。如果是 PRODUCE 生产请求，则将消息写入到底层的磁盘日志中；如果是 FETCH 请求，则从磁盘或页缓存中读取消息。

**IO 线程池处中的线程才是执行请求逻辑的线程**。Broker 端参数 `num.io.threads` 控制了这个线程池中的线程数。目前该参数默认值是 8，表示每台 Broker 启动后自动创建 8 个 IO 线程处理请求。

请求队列是所有网络线程共享的，而响应队列则是每个网络线程专属的。
我们再来看看刚刚的那张图，图中有一个叫 Purgatory 的组件，这是 Kafka 中著名的“炼狱”组件。它是用来缓存延时请求（Delayed Request）的。所谓延时请求，就是那些一时未满足条件不能立刻处理的请求。比如设置了 acks=all 的 PRODUCE 请求，一旦设置了 acks=all，那么该请求就必须等待 ISR 中所有副本都接收了消息后才能返回，此时处理该请求的 IO 线程就必须等待其他 Broker 的写入结果。当请求不能立刻处理时，它就会暂存在 Purgatory 中。稍后一旦满足了完成条件，IO 线程会继续处理该请求，并将 Response 放入对应网络线程的响应队列中。

# 10. Kafka控制器组件

**控制器组件（Controller），是 Apache Kafka 的核心组件。它的主要作用是在 Apache ZooKeeper 的帮助下管理和协调整个 Kafka 集群。**集群中任意一台 Broker 都能充当控制器的角色，但是，在运行过程中，只能有一个 Broker 成为控制器，行使其管理和协调的职责。换句话说，**每个正常运转的 Kafka 集群，在任意时刻都有且只有一个控制器**。官网上有个名为 `activeController` 的 JMX 指标，可以帮助我们实时监控控制器的存活状态。

**控制器是重度依赖 ZooKeeper 的，Apache ZooKeeper 是一个提供高可靠性的分布式协调服务框架。ZooKeeper 常被用来实现集群成员管理、分布式锁、领导者选举等功能。**它使用的数据模型类似于文件系统的树形结构，根目录也是以“/”开始。该结构上的每个节点被称为 znode，用来保存一些元数据协调信息。**znode 可分为持久性 znode 和临时 znode，临时节点会随着会话关闭删除。**

**ZooKeeper 赋予客户端监控 znode 变更的能力，即所谓的 Watch 通知功能**。一旦 znode 节点被创建、删除，子节点数量发生变化，抑或是 znode 所存的数据本身变更，ZooKeeper 会通过节点变更监听器 (ChangeHandler) 的方式显式通知客户端。

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2022.png](https://img.masaiqi.com/20210223101748.png)

- 如何选出控制器

    Broker 在启动时，会尝试去 ZooKeeper 中创建 `/controller` 节点。Kafka 当前选举控制器的规则是：**第一个成功创建 /controller 节点的 Broker 会被指定为控制器**。

- 控制器职责
    1. **主题管理（创建、删除、增加分区）**。执行`kafka-topics` 脚本时，大部分的后台工作都是控制器来完成的。
    2. **分区重分配**。执行`kafka-reassign-partitions` 脚本，对已有主题分区进行细粒度的分配功能。这部分功能也是控制器实现的。
    3. **Preferred 领导者选举**。Preferred 领导者选举主要是 Kafka 为了避免部分 Broker 负载过重而提供的一种换 Leader 的方案。
    4. **集群成员管理（新增 Broker、Broker 主动关闭、Broker 宕机）**

        控制器组件会利用 Watch 机制检查 ZooKeeper 的 `/brokers/ids` 节点下的子节点数量变更。目前，当有新 Broker 启动后，它会在 /brokers 下创建专属的 znode 节点。一旦创建完毕，ZooKeeper 会通过 Watch 机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化，进而开启后续的新增 Broker 作业。

        每个 Broker 启动后，会在 /brokers/ids 下创建一个临时 znode。当 Broker 宕机或主动关闭后，该 Broker 与 ZooKeeper 的会话结束，这个 znode 会被自动删除，控制器就可以通过Watch机制感知到并做善后处理。

    5. **数据服务**

        向其他 Broker 提供数据服务。控制器上保存了最全的集群元数据信息，其他所有 Broker 会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。

- 控制器保存的数据

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2023.png](https://img.masaiqi.com/20210223101749.png)

    比较重要的几个：

    1. 所有主题信息。包括具体的分区信息，比如领导者副本是谁，ISR 集合中有哪些副本等。
    2. 所有 Broker 信息。包括当前都有哪些运行中的 Broker，哪些正在关闭中的 Broker 等。
    3. 所有涉及运维任务的分区。包括当前正在进行 Preferred 领导者选举以及分区重分配的分区列表。
- 控制器故障转移（Failover）

    在 Kafka 集群运行过程中，只能有一台 Broker 充当控制器的角色，那么这就存在单点失效（Single Point of Failure）的风险，Kafka为控制器提供了故障转移功能（Failover）。

    流程如图：

    ![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2024.png](https://img.masaiqi.com/20210223101750.png)

    1. 最开始时，Broker 0 是控制器。当 Broker 0 宕机后，ZooKeeper 通过 Watch 机制感知到并删除了 `/controller` 临时节点。
    2. 所有存活的 Broker 开始竞选新的控制器身份。Broker 3 最终赢得了选举，成功地在 ZooKeeper 上重建了 `/controller` 节点。
    3. Broker 3 会从 ZooKeeper 中读取集群元数据信息，并初始化到自己的缓存中。至此，控制器的 Failover 完成。

**当你觉得控制器组件出现问题时**，比如主题无法删除了，或者重分区 hang 住了，不用重启 Kafka Broker 或控制器。**有一个简单快速的方式是，去 ZooKeeper 中手动删除 `/controller` 节点。具体命令是 `rmr /controller`。这样做的好处是，既可以引发控制器的重选举，又可以避免重启 Broker 导致的消息处理中断。**

# 11. 高水位和Leader Epoch

## 11.1. 什么是高水位

高水位（High Watermark，简称HW），一般定义为：在时刻 T，任意创建时间（Event Time）为 T’，且 T’≤T 的所有事件都已经到达或被观测到，那么 T 就被定义为水位。
**在 Kafka 的世界中，水位与时间无关。它是和位置信息绑定的，具体来说，它是用消息位移来表征的。**

## 11.2. Kafka高水位的作用

1. 定义消息可见性，即用来标识分区下的哪些消息是可以被消费者消费的。
2. 帮助 Kafka 完成副本同步。

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2025.png](https://img.masaiqi.com/20210223101751.png)

当 Kafka 的**若干个 Broker 成功地接收到一条消息并写入到日志文件后，这条消息就是“已提交消息”，**这里的具体多少个Broker(同分区多副本)写入成功取决于Broker端 `min.insync.replicas`参数值。

**位移值等于高水位的消息也属于未提交消息。也就是说，高水位上的消息是不能被消费者消费的。**

**日志末端位移，即 Log End Offset，简写是 LEO。它表示副本写入下一条消息的位移值，高水位值不会大于 LEO 值(小于等于)。**

**Kafka 使用 Leader 副本的高水位来定义所在分区的高水位。**

**实际理解起来，感觉就是双游标去管控多副本的进度**

- **LEO记录当前副本实际数据写入进度**
- **HW记录所有副本整体的写入进度**(根据各个副本的LEO进度有所差异，不大于当前副本LEO值)
- **HW不断位移追赶LEO。**

这里不讨论 Kafka 事务，因为事务机制会影响消费者所能看到的消息的范围，它不只是简单依赖高水位来判断。它依靠一个名为 LSO（Log Stable Offset）的位移值来判断事务型消费者的可见性。

## 11.3. 高水位更新机制

**每个副本对象都保存了一组高水位值和 LEO 值，在 Leader 副本所在的 Broker 上还保存了其他 Follower 副本的 LEO 值，这些副本又被称为远程副本（Remote Replica）。**

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2026.png](https://img.masaiqi.com/20210223101752.png)

Kafka 副本机制在运行过程中，会更新 Broker 1 上 Follower 副本的高水位和 LEO 值，同时也会更新 Broker 0 上 Leader 副本的高水位和 LEO 以及所有远程副本的 LEO，但**它不会更新远程副本的高水位值，也就是图中标记为灰色的部分**。

**Broker 0 上保存这些远程副本的主要作用是，帮助 Leader 副本确定其高水位，也就是分区高水位。**

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2027.png](https://img.masaiqi.com/20210223101753.png)

**“与 Leader 副本保持同步”，判断的条件有两个**：

1. 该远程 Follower 副本在 ISR 中。
2. 该远程 Follower 副本 LEO 值落后于 Leader 副本 LEO 值的时间，不超过 Broker 端参数 `replica.lag.time.max.ms` 的值。如果使用默认值的话，就是不超过 10 秒。

这俩条件好像在说一件事，但实际上如果是重新启动的副本，不在ISR中，但是满足ISR条件，分区HW值很有可能 > 该副本的LEO值，这是不被允许的。

Leader 副本处理生产者请求的逻辑如下：

1. 写入消息到本地磁盘。
2. 更新分区高水位值。
    1. 获取 Leader 副本所在 Broker 端保存的所有远程副本 LEO 值（LEO-1，LEO-2，……，LEO-n）。
    2. 获取 Leader 副本高水位值：currentHW。
    3. 更新 currentHW = max{currentHW, min（LEO-1, LEO-2, ……，LEO-n）}。

处理 Follower 副本拉取消息的逻辑如下：

1. 读取磁盘（或页缓存）中的消息数据。
2. 使用 Follower 副本发送请求中的位移值更新远程副本 LEO 值。
3. 更新分区高水位值（具体步骤与处理生产者请求的步骤相同）。Follower 副本从 Leader 拉取消息的处理逻辑如下：写入消息到本地磁盘。更新 LEO 值。更新高水位值。
    1. 获取 Leader 发送的高水位值：currentHW。
    2. 获取步骤 2 中更新过的 LEO 值：currentLEO。
    3. 更新高水位为 min(currentHW, currentLEO)。

## 11.4. 副本同步机制

首先是初始状态。下面这张图中的 remote LEO 就是刚才的远程副本的 LEO 值。在初始状态时，所有值都是 0。

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2028.png](https://img.masaiqi.com/20210223101754.png)

当生产者给主题分区发送一条消息后，Leader副本写入磁盘，Leader的LEO值变为1，Follower 再次尝试从 Leader 拉取消息，Follower的LEO值也变为1：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2029.png](https://img.masaiqi.com/20210223101755.png)

此时，Leader 和 Follower 副本的 LEO 都是 1，但各自的高水位依然是 0，还没有被更新。**它们需要在下一轮的拉取中被更新**：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2030.png](https://img.masaiqi.com/20210223101756.png)

在新一轮的拉取请求中，由于位移值是 0 的消息已经拉取成功，因此 Follower 副本这次请求拉取的是位移值 =1 的消息。

1. Leader 副本接收到此请求后，更新远程副本 LEO 为 1，然后更新 Leader 高水位为 1(前文图中分区HW更新的第二个时机)。
2. Leader将当前已更新过的高水位值 1 发送给 Follower 副本。Follower 副本接收到以后，也将自己的高水位值更新成 1。至此，一次完整的消息同步周期就结束了。**事实上，Kafka 就是利用这样的机制，实现了 Leader 和 Follower 副本之间的同步**。

## 11.5. Leader Epoch机制

Leader 副本高水位更新和 Follower 副本高水位更新在时间上是存在错配的，这种错配是很多“数据丢失”或“数据不一致”问题的根源，因此就有了Leader Epoch。

**Leader Epoch，大致可以认为是 Leader 版本。**它由两部分数据组成：

1. **Epoch。一个单调增加的版本号**。每当副本领导权发生变更时，都会增加该版本号。小版本号的 Leader 被认为是过期 Leader，不能再行使 Leader 权力。
2. **起始位移（Start Offset）**。Leader 副本在该 Epoch 值上写入的首条消息的位移。

比如Leader A有Leader Epoch<0, 0>，代表版本号0，起始位移0，写入到位移20后，发生了Leader切换，切换到了B，则会写入一个新的Leader Epoch<1, 20>，版本号1，起始位移20。

Kafka Broker 会在内存中为每个分区都缓存 Leader Epoch 数据，同时它还会定期地将这些信息持久化到一个 checkpoint 文件中，查看任意一个Topic的日志文件夹中可以看到：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2031.png](https://img.masaiqi.com/20210223101757.png)

当 Leader 副本写入消息到磁盘时，Broker 会尝试更新这部分缓存。

1. 如果该 Leader 是首次写入消息，那么 Broker 会向缓存中增加一个 Leader Epoch 条目，
2. 否则就不做更新。

这样，每次有 Leader 变更时，新的 Leader 副本会查询这部分缓存，取出对应的 Leader Epoch 的起始位移，以避免数据丢失和不一致的情况。

举个例子，在Broker 端参数 `min.insync.replicas` 设置为 1，未引入Leader Epoch机制时：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2032.png](https://img.masaiqi.com/20210223101758.png)

图中副本A和副本B的HW分别为2和1，这在副本同步机制中是可能发生的，Leader副本可能正在准备把自己的HW(分区HW)发送给Follower副本，这时，如果Follower副本B宕机，**重启后副本 B 会执行日志截断操作，将 LEO 值调整为之前的高水位值**，也就是 ，这样大于等于高水位值的日志就会被删除，就只有位移0的数据了。如果副本A宕机，副本B则成为了Leader，之后副本A重启后会同步副本B的数据，这样就永远丢失了位移1的数据。

引入Leader Epoch后：

![Kafka%E5%9F%BA%E7%A1%80%2059767abf68e24c1daeb518e5ca64fd2b/Untitled%2033.png](https://img.masaiqi.com/20210223101759.png)

Follower 副本 B 重启回来后，需要向 A 发送一个特殊的请求去获取 Leader 的 LEO 值。在这个例子中，该值为 2。当获知到 Leader LEO=2 后，**B 发现该 LEO 值不比它自己的 LEO 值小，而且缓存中也没有保存任何起始位移值 > 2 的 Epoch 条目，因此 B 无需执行任何日志截断操作**。这是对高水位机制的一个明显改进，即**副本是否执行日志截断不再依赖于高水位进行判断**。

当 A 重启回来后，执行与 B 相同的逻辑判断，发现也不用执行日志截断，至此位移值为 1 的那条消息在两个副本中均得到保留。后面当生产者程序向 B 写入新消息时，副本 B 所在的 Broker 缓存中，会生成新的 Leader Epoch 条目：[Epoch=1, Offset=2]。之后，副本 B 会使用这个条目帮助判断后续是否执行日志截断操作。这样，**通过 Leader Epoch 机制，形成了Leader副本的版本号-位移数据，避免了数据丢失**。

# 12. 参考资料

- [Kafka Doc](https://kafka.apache.org/documentation/#upgrade_11_exactly_once_semantics)
- [Kafka-transactions-Chrząszcz's blog](https://chrzaszcz.dev/2019/12/kafka-transactions/)