---
title: "基于Redis实现分布式锁与Redis的ACID特性(以秒杀业务为例)"
date: 2021-01-26 22:06:06.842
draft: false
type: "post"
showTableOfContents: true
tags: ["Redis","NoSQL"]
---

# 基于Redis实现分布式锁与Redis的ACID特性(以秒杀业务为例)

# 1. 准备

Redis实现秒杀业务，主要需要三块的知识储备：

- 秒杀业务流程
- 如何保证原子性(Redis能为我们做哪些ACID保证)
- Redis实现一个分布式锁

下文会围绕这3块进行展开叙述。

# 2. 秒杀业务流程

分为三个阶段，秒杀阶段前，中，后。

**秒杀阶段前**，通常用户会不断刷新商品详情页，这会导致详情页的瞬时请求量剧增。**一般是尽量把商品详情页的页面元素静态化，然后使用 CDN 或是浏览器把这些静态化的元素缓存起来。**

**秒杀阶段中**，这是我们重点需要讨论的流程，重点是库存的扣减，这部分包括“查库存→扣库存→刷新缓存”三个环节，需要保证**原子性**。除此以外，这部分也会有很高的并发量，不能直接交给数据库，避免数据库压力过大，需要一个支持**高并发的系统**。

**秒杀阶段后**，可能还会有部分用户刷新商品详情页，尝试等待有其他用户退单。而已经成功下单的用户会刷新订单详情，跟踪订单的进展。这个阶段请求已经下降很多，不做额外处理。

# 3. Redis为什么适合秒杀业务

- **支持高并发**。这个很简单，Redis 本身高速处理请求的特性就可以支持高并发。而且，如果有多个秒杀商品，我们也可以使用切片集群，用不同的实例保存不同商品的库存，这样就避免，使用单个实例导致所有的秒杀请求都集中在一个实例上的问题了。
- 保证库存查验和库存扣减**原子性执行**。针对这条要求，我们就可以使用 Redis 的原子操作或是分布式锁这两个功能特性来支撑了。

# 4. Redis事务

## 4.1. Redis事务的使用

Redis 提供了 MULTI、EXEC 两个命令来实现事务。

```bash
#开启事务
127.0.0.1:6379> MULTI
OK
#需要事务的命令，压入事务队列
127.0.0.1:6379> ......
#实际执行事务
127.0.0.1:6379> EXEC
```

## 4.2. Redis事务的ACID特性

ACID包括

- 原子性（Atomicity）
- 一致性（Consistency）
- 隔离性（Isolation）
- 持久性（Durability）

### 4.2.1. Redis事务的原子性(Atomicity)

Redis的事务的原子性和Mysql事务原子性不太一样，因为**Redis本质上不支持回滚操作**，这一点需要特别注意，尽管提供了**`discard`命令，但其实这个命令只是用来清空当前事务队列，取消事务**。

- **命令入队时就报错，会放弃事务执行，保证原子性。**

```bash
#开启事务
127.0.0.1:6379> MULTI
OK
#发送事务中的第一个操作，但是Redis不支持该命令，返回报错信息
127.0.0.1:6379> PUT a:stock 5
(error) ERR unknown command `PUT`, with args beginning with: `a:stock`, `5`, 
#发送事务中的第二个操作，这个操作是正确的命令，Redis把该命令入队
127.0.0.1:6379> DECR b:stock
QUEUED
#实际执行事务，但是之前命令有错误，所以Redis拒绝执行
127.0.0.1:6379> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
```

- **命令入队时没报错，实际执行时报错，不保证原子性。**如果压入事务队列的命令语法上没问题，执行时出错，Redis会把错误信息抛出，并把正确的命令执行完。

```bash
#开启事务
127.0.0.1:6379> MULTI
OK
#发送事务中的第一个操作，LPOP命令操作的数据类型不匹配，此时并不报错
127.0.0.1:6379> LPOP a:stock
QUEUED
#发送事务中的第二个操作
127.0.0.1:6379> DECR b:stock
QUEUED
#实际执行事务，事务第一个操作执行报错
127.0.0.1:6379> EXEC
1) (error) WRONGTYPE Operation against a key holding the wrong kind of value
2) (integer) 8
```

- **EXEC 命令执行时实例故障，如果开启了 AOF 日志，可以保证原子性。**

### 4.2.2. Redis事务的一致性(Consistency)

- 命令入队时就报错在这种情况下，事务本身就会被放弃执行，所以可以保证数据库的一致性。
- 命令入队时没报错，实际执行时报错在这种情况下，有错误的命令不会被执行，正确的命令可以正常执行，也不会改变数据库的一致性。
- EXEC 命令执行时实例发生故障在这种情况下，实例故障后会进行重启，这就和数据恢复的方式有关了，我们要根据实例是否开启了 RDB 或 AOF 来分情况讨论：
    1. 没有启用RDB和AOF，数据都没有了，数据库是一致的。
    2. 开启了RDB，**RDB 快照不会在事务执行时执行**，所以，事务命令操作的结果不会被保存到 RDB 快照中，使用 RDB 快照进行恢复时，数据库里的数据也是一致的。
    3. 开启了AOF，而事务操作还没有被记录到 AOF 日志时，实例就发生了故障，那么，使用 AOF 日志恢复的数据库数据是一致的。如果只有部分操作被记录到了 AOF 日志，我们可以使用 redis-check-aof 清除事务中已经完成的操作，数据库恢复后也是一致的。

### 4.2.3. Redis事务的隔离性(Isolation)

- 并发操作在 EXEC 命令前执行，此时，**隔离性的保证要使用 WATCH 机制来实现**，否则隔离性无法保证；(当事务调用 EXEC 命令执行时，WATCH 机制会先检查监控的键是否被其它客户端修改了。如果修改了，就放弃事务执行，避免事务的隔离性被破坏。)

    ![%E5%9F%BA%E4%BA%8ERedis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E4%B8%8ERedis%E7%9A%84ACID%E7%89%B9%E6%80%A7(%E4%BB%A5%E7%A7%92%E6%9D%80%E4%B8%9A%E5%8A%A1%E4%B8%BA%E4%BE%8B)%208b7af5c985824282909c8174f8d99752/Untitled.png](https://img.masaiqi.com/20210126220523.png)

- 并发操作在 EXEC 命令后执行，此时，隔离性可以保证。Redis单线程，会保证事务的命令执行完再执行其他命令。

### 4.2.4. Redis事务的持久性(Durability)

**不管 Redis 采用什么持久化模式，事务的持久性属性是得不到保证的。**

- 没有使用 RDB 或 AOF，那么事务的持久化属性肯定得不到保证。
- 开启了 RDB，那么，在一个事务执行后，而下一次的 RDB 快照还未执行前，如果发生了实例宕机，这种情况下，事务修改的数据也是不能保证持久化的。
- 开启了AOF，因为 AOF 模式的三种配置选项 no、everysec 和 always 都会存在数据丢失的情况，所以，事务的持久性属性也还是得不到保证。

# 5. 其他方式保证Redis原子性

## 5.1. 单命令

比如**`INCR/DECR`**这种自增自减的单命令，天生就具有原子性，无需额外操作。缺点就是业务范围局限与自增自减少

## 5.2. lua脚本

上文中提到，Redis的事务拥有ACID的特性，因此确实可以用Redis事务来保证一定的原子性。除此以外，用lua脚本事实上也可以实现原子性。

具体来说就是**将临界区代码写成一个lua脚本，交给Redis执行**。

这里假设我有一个RMW(Read-Modify-Write)的业务需求，读取key为“lua”的String类型数据，如果
值为一个指定3，就删除它，否则不变，看一下Redis里如何操作，笔者这里是写了一个通用的redis脚本。

![%E5%9F%BA%E4%BA%8ERedis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E4%B8%8ERedis%E7%9A%84ACID%E7%89%B9%E6%80%A7(%E4%BB%A5%E7%A7%92%E6%9D%80%E4%B8%9A%E5%8A%A1%E4%B8%BA%E4%BE%8B)%208b7af5c985824282909c8174f8d99752/Untitled%201.png](https://img.masaiqi.com/20210126220524.png)

当然，这里有个问题，在于每次都要传输lua脚本的数据，会带来额外的网络IO开销，我们可以通过`SCRIPT LOAD`命令把lua脚本存在redis服务器中，这个命令会返回一个sha值，我们通过`EVALSHA`命令用sha值执行脚本：

![%E5%9F%BA%E4%BA%8ERedis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E4%B8%8ERedis%E7%9A%84ACID%E7%89%B9%E6%80%A7(%E4%BB%A5%E7%A7%92%E6%9D%80%E4%B8%9A%E5%8A%A1%E4%B8%BA%E4%BE%8B)%208b7af5c985824282909c8174f8d99752/Untitled%202.png](https://img.masaiqi.com/20210126220525.png)

# 6. 基于单机Redis实现一个分布式锁

笔者这里提供一份基于Spring IOC + Redis + lua + Jedis实现的分布式锁，实现了JDK的标准Lock接口(省略了Jedis的获取和关闭工具类，这个各家公司都有封装，比较基础，不负赘述)：

```java
import lombok.extern.slf4j.Slf4j;
import org.hibernate.validator.constraints.NotBlank;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.util.Assert;
import org.springframework.util.StringUtils;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.params.SetParams;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;

import static org.springframework.beans.factory.config.ConfigurableBeanFactory.SCOPE_PROTOTYPE;

/**
 * 基于单Redis节点实现的分布式锁
 * 注意，这里作用域为prototype，即原型模式，非单例。
 *
 * @author masaiqi
 * @date 2021/1/22 9:49 上午
 */
@Component
@Scope(SCOPE_PROTOTYPE)
@Slf4j
public class DistributedLockSingleRedis extends AbstractDistributedLock implements DistributedLock {

	/**
	 * 过期时间(second)
	 */
	private Integer expireTime;

	/**
	 * 删除文件的lua脚本对应的sha值
	 */
	@Value("${lua-sha.redis-lock}")
	private String delKeySha;

	/**
	 * 默认过期时间(second) 半小时
	 */
	private Integer defaultExpireTime = 30 * 60;

	/**
	 * 初始化状态
	 * 多次调用只生效一次
	 *
	 * @param expireTime 过期时间
	 * @return void
	 * @author sq.ma
	 * @date 2021/1/22 12:01 下午
	 */
	public void initState(@NotBlank String lockKey, @NotBlank String clientId, Integer expireTime) {
		if (StringUtils.hasText(lockKey) && this.lockKey == null) {
			this.lockKey = lockKey;
		}
		if (StringUtils.hasText(clientId) && this.clientId == null) {
			this.clientId = clientId;
		}
		if (this.expireTime == null) {
			//添加随机时间(0 ～ 2 分钟)，避免大量缓存同时失效导致缓存雪崩
			int randomNum = (int) (Math.random() * 60 * 2);
			if (expireTime != null) {
				this.expireTime = expireTime + randomNum;
			} else {
				//认为必传 否则可能造成锁永远无法释放(客户端宕机) 所以这里设置默认值。
				this.expireTime = this.defaultExpireTime + randomNum;
			}
		}
	}

	/**
	 * 阻塞式获取锁，一直堵塞直到成功拿到锁
	 */
	@Override
	public void lock() {
		Jedis jedis = null;
		try {
			jedis = RedisUtil.getJedisResource();
			while (setKey(jedis) == null) {
				// Do nothing. 结果为空 -> 没有获取到锁 -> 一直循环到获取到锁为止
			}
			log.info(String.format("[Key: %s | Value: %s] gets DistributedLockSingleRedis Lock", lockKey, clientId));
		} finally {
			RedisUtil.closeJedisWithLogging("DistributedLockSingleRedis Lock", jedis);
		}
	}

	/**
	 * 阻塞式获取锁，响应线程中断信号，如果发生中断立刻停止获取锁并检查并释放锁
	 *
	 * @throws InterruptedException
	 */
	@Override
	public void lockInterruptibly() throws InterruptedException {
		if (Thread.interrupted()) {
			throw new InterruptedException();
		}
		Jedis jedis = null;
		try {
			jedis = RedisUtil.getJedisResource();
			while (setKey(jedis) == null) {
				if (Thread.interrupted()) {
					throw new InterruptedException();
				}
			}
			log.info(String.format("[Key: %s | Value: %s] gets DistributedLockSingleRedis Lock", lockKey, clientId));
		} finally {
			RedisUtil.closeJedisWithLogging("DistributedLockSingleRedis Lock", jedis);
			unlock();
		}
	}

	/**
	 * 非阻塞式获取锁
	 */
	@Override
	public boolean tryLock() {
		Jedis jedis = null;
		Boolean result = false;
		try {
			jedis = RedisUtil.getJedisResource();
			if (setKey(jedis) != null) {
				log.info(String.format("[Key: %s | Value: %s] gets DistributedLockSingleRedis Lock", lockKey, clientId));
				result = true;
			}
		} finally {
			RedisUtil.closeJedisWithLogging("DistributedLockSingleRedis Lock", jedis);
		}
		return result;
	}

	/**
	 * 阻塞式获取锁,直到超时或者获取到锁后返回
	 */
	@Override
	public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
		long tryTime = unit.toMillis(time);
		long hasTriedTime = 0;

		Jedis jedis = null;
		Boolean result = false;
		try {
			jedis = RedisUtil.getJedisResource();
			while (hasTriedTime <= tryTime) {
				long startTime = System.currentTimeMillis();
				String setKeyResult = setKey(jedis);
				if (setKeyResult != null) {
					log.info(String.format("[Key: %s | Value: %s] gets DistributedLockSingleRedis Lock", lockKey, clientId));
					result = true;
				}
				long endTime = System.currentTimeMillis();
				hasTriedTime = endTime - startTime + hasTriedTime;
			}
		} finally {
			RedisUtil.closeJedisWithLogging("DistributedLockSingleRedis Lock", jedis);
		}
		return result;
	}

	/**
	 * 释放锁
	 */
	@Override
	public void unlock() {
		Jedis jedis = null;
		try {
			jedis = RedisUtil.getJedisResource();
			// 执行lua脚本，对应resources/redis-lua/redis_lock_lua.script这个文件
			Long result = (Long) jedis.evalsha(delKeySha, 1, lockKey, clientId);
			if (result == 1L) {
				log.info(String.format("[Key: %s | Value: %s] releases DistributedLockSingleRedis Lock", lockKey, clientId));
			}
		} finally {
			RedisUtil.closeJedisWithLogging("DistributedLockSingleRedis Lock", jedis);
		}
	}

	@Override
	public Condition newCondition() {
		throw new RuntimeException("暂不支持!");
	}

	/**
	 * 状态检查，请调用{@link #initState(String, String, Integer)} 进行初始化。
	 */
	@Override
	public void checkState() {
		super.checkState();
		Assert.notNull(expireTime, "您必须传入过期时间!");
		Assert.hasText(delKeySha, "您必须传入sha!");
	}

	/**
	 * 设置redis值
	 *
	 * @return
	 */
	private String setKey(Jedis jedis) {
		checkState();
		// Set key if not exists -> execute command 'SET key value [EX seconds | PX milliseconds]  [NX]'
		return jedis.set(lockKey, clientId, SetParams.setParams().ex(expireTime).nx());
	}

}
```

代码中用到的lua脚本：

```lua
if redis.call("get",KEYS[1]) == ARGV[1] then 
	return redis.call("del",KEYS[1]) 
else 
	return 0;
end;
```

下面具体说一下思路。

首先，分布式锁的核心是维护一个共享字段，也就是笔者上面代码的的**lockKey**，有几个注意点：

- 因为需要“由什么客户端开启锁，由什么客户端关闭锁”，因此值用**clientId**，可以是客户端的唯一标志：
    1. 加锁只需要`set nx` 即可，如果key已经有值，这个命令不会返回结果，这样同时刻只会有一个客户端加锁成功，并将自己的clientId赋为值。
    2. 释放锁需要查询当前锁的值是否为当前客户端，是则删除这个字段。这里有RMW操作，因此再用一个lua脚本来保证原子性。
    3. 设置一个过期时间，避免客户端宕机导致锁永远无法释放；过期时间增加随机值，避免同时过期带来缓存雪崩；过期时间设置到具体什么时间，避免过期时间带来的主从不一致。

# 7. 基于多个 Redis 节点实现高可靠的分布式锁

使用**Redlock 算法**。

**基本思路，是让客户端和多个独立的 Redis 实例依次请求加锁，如果客户端能够和半数以上的实例成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁了，否则加锁失败。**

具体来说：

1. 客户端获取当前时间。
2. 客户端按顺序依次向 N 个 Redis 实例执行加锁操作。这里的加锁操作和在单实例上执行的加锁操作一样，使用 SET 命令，带上 NX，EX/PX 选项，以及带上客户端的唯一标识。当然，如果某个 Redis 实例发生故障了，为了保证在这种情况下，Redlock 算法能够继续运行，我们需要给加锁操作设置一个超时时间。如果客户端在和一个 Redis 实例请求加锁时，一直到超时都没有成功，那么此时，客户端会和下一个 Redis 实例继续请求加锁。加锁操作的超时时间需要远远地小于锁的有效时间，一般也就是设置为几十毫秒。
3. 一旦客户端完成了和所有 Redis 实例的加锁操作，客户端就要计算整个加锁过程的总耗时。

**客户端只有在满足下面的这两个条件时，才能认为是加锁成功：**

- **客户端从超过半数（大于等于 N/2+1）的 Redis 实例上成功获取到了锁**
- **客户端获取锁的总耗时没有超过锁的有效时间。**

# 8. 回到秒杀业务

到这边需要的工具基本齐全了，基本实现思路就是将库存维护到Redis中，然后并发的维护这个key-value数据，其中需要保证原子性，我们有以下方式

- Redis事务
- lua脚本
- 分布式锁

其中，可以使用切片集群中的不同实例来分别保存分布式锁和商品库存信息来增加吞吐量。
![image-20210107190944577](https://img.masaiqi.com/20210107190947.png)