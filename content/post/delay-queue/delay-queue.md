---
title: "基于DelayedQueue实现订单延迟关闭/超时取消"
description: base on Delay Queue achieve orders cancel out of time 
date: 2026-06-12T09:31:27+08:00
categories:
    - 延迟队列
    - 分布式
tags:
    - Redis
    - Redisson
    - DelayedQueue
image: 
math: 
license: 
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list

---



## 基于DelayedQueue实现订单延迟关闭/超时取消

### 没有延迟队列如何实现？

定时扫描数据库

每个未支付的订单放入到队列中，一旦过期则出队。如何判断过期？

1. 定时任务扫描队列，可能导致订单超时还未关闭。

2. 触发式，一旦订单过期则notify唤醒某个线程去处理订单关闭。

基于消息队列RocketMQ, 支持延迟消息，消息在指定的延迟时间后被投递到目标队列。

基于Redis的Stream

基于Redisson的RDelayedQueue

RabbitMQ的延迟插件或者Kafka的时间轮

### 方案比对

消息队列需要额外引入组件...

-----

## RDelayedQueue的原理

### 内部数据结构

分为三个队列

* 【消息延时队列】`Zset`类型，利用按照到期时间排序的特性，可以很快找到下一个要到期的消息，客户端内部自己定时到【消息目标队列】取
* 【消息顺序队列】这个队列对分析的流程关联不大，可以忽略
* 【消息目标队列】`List` 类型，存放到期的消息，供消费端取

【消息延时队列】队列里存的时间（也就是 zet 的 score）是到期的时间戳

**把到期的消息从【消息延时队列】移到【消息目标队列】里** ，这句话实际的代码逻辑是这样：把【消息延时队列】和【消息顺序队列】里的到期消息移除，把它们插入到【消息目标队列】

### 基本流程

先说**发送延迟消息** ，发送的延迟消息会先存在【消息延时队列】和【消息顺序队列】，如果【消息延时队列】原本是空的，会发布订阅信息提醒有新的消息。

**获取延迟消息** 只需要从【消息目标队列】阻塞的取就行了，因为里面都是到期数据。



#### 怎么样判断时间到了，把【消息延时队列】里的消息移动到【消息目标队列】里呢？

这部分工作交给了**初始化延时队列** 来处理。

这里面会定时从【消息延时队列】查询最新到期时间，定时去把【消息延时队列】里的消息移动到【消息目标队列】里。

如果【消息延时队列】是空的，就不会再定时查，而是等待发布订阅信息提醒，再定时把【消息延时队列】里的消息移动到【消息目标队列】里。



### 发送延迟队列的消息

#### 源码解析

```java
public void produce() {
  String queuename = "delay-queue";
  RBlockingQueue<String> blockingQueue = redissonClient.getBlockingQueue(queuename);
  RDelayedQueue<String> delayedQueue = redissonClient.getDelayedQueue(blockingQueue);
  delayedQueue.offer("测试延迟消息", 5, TimeUnit.SECONDS);
}
```

`RedissonDelayedQueue.offer()`底层调用`RedissonDelayedQueue.offerAsync`)

![image](3-1.png)

```java
public RFuture<Void> offerAsync(V e, long delay, TimeUnit timeUnit) {
        if (delay < 0) {
            throw new IllegalArgumentException("Delay can't be negative");
        }

        long delayInMs = timeUnit.toMillis(delay);
        long timeout = System.currentTimeMillis() + delayInMs;

        byte[] random = getServiceManager().generateIdArray(8);
        return commandExecutor.evalWriteNoRetryAsync(getRawName(), codec, RedisCommands.EVAL_VOID,
                "local value = struct.pack('Bc0Lc0', string.len(ARGV[2]), ARGV[2], string.len(ARGV[3]), ARGV[3]);"
              + "redis.call('zadd', KEYS[2], ARGV[1], value);"
              + "redis.call('rpush', KEYS[3], value);"
              // if new object added to queue head when publish its startTime 
              // to all scheduler workers 
              + "local v = redis.call('zrange', KEYS[2], 0, 0); "
              + "if v[1] == value then "
                 + "redis.call('publish', KEYS[4], ARGV[1]); "
              + "end;",
              Arrays.asList(getRawName(), timeoutSetName, queueName, channelName),
              timeout, random, encode(e));
    }
```

- `timeout`: 当前时间+延迟时间，作为`ZSet`的`score`

- `random`:因为延迟队列中可能会有相同的元素,随机 ID 保证了 ZSet 和 List 中元素的唯一性

- Key的映射
  
  * `KEYS[1]`: `getRawName()` (原队列名，Lua脚本中未直接使用，但作为上下文传入)
  * `KEYS[2]`: `timeoutSetName` (存放延迟时间的 ZSet)
  * `KEYS[3]`: `queueName` (存放实际元素的 List)
  * `KEYS[4]`: `channelName` (用于发布通知的 Pub/Sub 频道)

- `redis.call('zadd', KEYS[2], ARGV[1], value);`  将 `value` 添加到 `timeoutSetName`（有序集合）中，分数（score）为前面计算的 `timeout`（过期时间戳）。

- `redis.call('rpush', KEYS[3], value);`  同时将 `value` 追加到 `queueName`（列表）的尾部

- `redis.call('zrange', KEYS[2], 0, 0);`获取 ZSet 中分数最小（即最早到期）的元素

- `if v[1] == value then redis.call('publish', KEYS[4], ARGV[1]);`当前插入的消息是最近到期消息，则让订阅者获取到期时间

Lua脚本的内容总结为

* **将消息和到期时间插入【消息延时队列】和【消息顺序队列】**

* **如果最近到期的消息是刚刚插入的消息，则对指定主题发布到期时间，目的是为了让客户端定时去把【消息延时队列】里的到期数据移动到【消息目标队列】**

```
若消息延迟队列里没有消息，则不会定时查该队列；若新添加了消息，则会publish，发布有新的消息订阅，则消费者就会定时去查该队列；若新添加的消息是最新的消息，则发布新的消息订阅，更新最早过期时间，改变定时查询时间。
```



#### ZSet增加的唤醒机制

* **唤醒机制**：正如您提供的代码所示，当新插入的任务的到期时间比 ZSet 中现有所有任务的到期时间都早时，它会成为 ZSet 的第一个元素。此时，调度器需要立刻被唤醒（通过 Pub/Sub），否则调度器还在傻等之前的“最早时间”，导致这个更高优先级的任务被延迟处理。
  
  

#### 为什么需要zset和list两个数据结构配合使用

ZSet，能解决按时间排序和到期检测的问题，但在消费时存在致命缺陷：ZSet 的 `zpopmin` 等弹出操作在 Redis 早期版本中不是原子的（需要先查后删），在分布式并发环境下，极易导致**同一个任务被多个消费者同时获取（重复消费）**

当 ZSet 中有任务到期时，将任务从 ZSet 中移除，通过 `RPUSH`操作，将任务**转移**到另一个目标 List 中

消费者直接使用 `BLPOP`（阻塞弹出）从目标 List 中获取任务。`BLPOP` 是 Redis 原生支持的原子操作，天然保证了**一个任务只会被一个消费者获取**，彻底避免了并发竞争和重复消费的问题。



#### evalWriteNoRetryAsync

方法名中带有 `NoRetry`，意味着如果 Redis 集群发生主从切换或连接异常导致该脚本执行失败，Redisson **不会自动重试**。因为如果重试，可能会导致消息被重复插入（幂等性问题）。调用方需要自行处理这次异步操作的异常结果。

#### Pub/Sub 的网络抖动风险：

当插入了一个最早到期的任务时，会通过 `publish` 通知调度器。如果此时 Redis 调度器与 Redis 之间的网络断开，可能会丢失这条通知。不过 Redisson 有兜底机制：调度器除了监听 Pub/Sub，通常还会有一个后台定时任务（如每秒）去轮询 ZSet，所以即使通知丢失，任务也只会延迟触发，不会漏触发。







### 获取延迟队列的消息

```java
public void consume() throws InterruptedException {
  String queuename = "delay-queue";
  RBlockingQueue<String> blockingQueue = redissonClient.getBlockingQueue(queuename);
  RDelayedQueue<String> delayedQueue = redissonClient.getDelayedQueue(blockingQueue);
  String msg = blockingQueue.take();
  //收到消息进行处理
...
}
```

`blockingQueue.take();`底层调用的是`RedissonBlockingQueue.takeAsync()`

```java
public RFuture<V> takeAsync() {
        return commandExecutor.writeAsync(getRawName(), codec, RedisCommands.BLPOP_VALUE, getRawName(), 0);
    }
```

可以看出**其实只是对【消息目标队列】执行 blpop 阻塞的获取到期消息**（`blpop`从List中移除并返回第一个元素的阻塞命令）

### 初始化延迟队列





----

## 基于延迟队列实现订单延迟关闭/超时取消的实现





----

## 其他问题

### 其他应用场景

* **订单超时自动取消**：下单后 30 分钟未支付，自动取消订单并释放库存。
* **延迟消息推送**：用户注册成功后，延迟 24 小时发送关怀邮件。
* **重试机制**：任务执行失败后，延迟 5 秒再进行重试。
* **限时优惠**：活动到指定时间后自动结束。

### 消息挤压，将Redis撑爆



### 防重复消费



----

## 参考

[开发实战：使用Redisson实现分布式延时消息，订单30分钟关闭的另外一种实现！-阿里云开发者社区](https://developer.aliyun.com/article/1627048)

[基于Redisson的高性能延迟队列架构设计与实现从去年到现在，这套延迟队列方案已经在生产环境稳定运行了一年多。期间经历 - 掘金](https://juejin.cn/post/7535356123886239784#heading-32)

[基于redis,redisson的延迟队列实践 - 字节悦动 - 博客园](https://www.cnblogs.com/better-farther-world2099/articles/15216447.html)







持久化，不存在丢失





消息队列特征：解耦，异步，削峰填谷

RocketMQ为了保证消息的可靠性传递，可能会多次发送消息，所以消费方要做好幂等性的保护措施
