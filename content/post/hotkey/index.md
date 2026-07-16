---
title: "Hotkey应用"
description: 
date: 2026-07-16T21:18:50+08:00
image: 
math: 
license: 
comments: true
draft: true
build:
    list: always    # Change to "never" to hide the page from the list
---

### 热Key探测的场景

考虑到用户访问量的增加，对系统性能和稳定性要求变高，为了减少用户加载题目的时间，对经常访问的数据进行缓存。

对于可以预判到的热点数据可以直接进行缓存预热，比如首页数据、排行榜题目/题库、投放广告的数据。

但是对于一些无法预测的数据，如果还没缓存，就突然被大量请求访问，系统就会被压垮。

对于以上情况，系统需要做到**自动发现热点**，做多级缓存来顶住大流量访问的压力。(当然通常没有大量用户会访问，但是可能存在大量网络攻击)

### HotKey的配置流程

启动etcd

启动worker

打包Client,给项目引用

配置HotKey的参数，启动项目时，后台会加载对应配置，`starter.startPipeline`启动一个线程去检测Key，推送Key

### 热key的定义

一定时间内对某个Key请求量大于某个阈值

### 动态检测热点Key的实现方案

基于**热点探测+二级缓存**方案，核心思路是：**用切面无侵入地封装缓存逻辑，结合京东开源的HotKey框架动态监测热Key，热Key数据自动缓存在JVM本地（Caffeine），冷Key走Redis**，最终解决了高并发场景下Redis单分片的性能瓶颈。



热Key的QPS从Redis的5万提升到了本地缓存的20万+，Redis的CPU使用率下降了60%

![img](https://cdn.nlark.com/yuque/0/2026/png/40619548/1775975572968-6ae3604f-c291-40fc-a6f1-6babd3c7ecd4.png)

Etcd：充当注册中心和配置中心

Worder集群：统计 流量次数

![img](https://cdn.nlark.com/yuque/0/2026/png/40619548/1775979836564-3102c643-d404-476c-970f-5a51f5d7b984.png)





### 项目架构图和二级缓存图

![img](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776099856975-ab53af3b-f51e-483f-bcc0-ea7514706c98.png)

![img](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776100069513-19cd43aa-855d-441c-a4f6-07b4509a54a6.png)



### HotKey的jar包中的代码逻辑?

1. 业务调用 JdHotKeyStore.isHotKey(key) 或 getValue(key)
            ↓
2. Client端把key上报给Worker（每500ms批量发送一次）
            ↓
3. Worker用滑动窗口累加计算频率，达到阈值后判定为热key
            ↓
4. Worker把这个key推送给所有Client端（告诉所有人：这个key是热的）
            ↓
5. Client端收到后，在本地Caffeine缓存中标记这个key是热的
            ↓
6. 业务代码需要自己从Redis/DB获取value，再通过smartSet()塞进去

### **高性能的数据上报：如何高效地发送海量待测key？**

客户端不会每次判断都发一次网络请求，而是用了一个非常聪明的**聚合策略**：

- **本地聚合**：所有要探测的key会先在一个本地队列里短暂停留。
- **定时批量上报**：默认每 500 毫秒，把这段时间内收集到的所有 key 打包成一个“小包裹”，一次性发给 worker 集群。(key通过Hash发送到一个worker中，统计key的访问次数)

这种做法极大地减少了网络开销，是它能支撑超高吞吐量的关键技术之一





### HotKey里的worker如何统计Key的访问次数是否超过rule的阈值



### 将热Key推送到Client

设置到本地缓存但是value为null ，通过判断是否是热Key, 使用smartSet(key,value)才会设置本地缓存的value值。



### 热Key会自动续期吗？如果是热Key了，还会继续推送访问次数到worker吗？

![img](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776095806922-5bb49356-0a13-423c-a495-3ad3188dd745.png)

![img](https://cdn.nlark.com/yuque/0/2026/png/40619548/1776095906823-df454443-9c86-47d7-8379-8b35f23681cc.png)

如果不是热Key，则会上报；

是热Key，则会先获取热Key的过期时间，判断如果还有两秒快过期则会继续上报。(如果在这两秒内还是高频的Key则会进行续期)



### 二级缓存的过期时间设置

1. **本地缓存 (JVM/Caffeine): 建议5分钟+随机偏移量（防止缓存雪崩）**

- **理由**: 本地缓存是为了在极短时间内（如大促流量峰值）扛住超高并发。它应该“快进快出”，让热点数据能被快速刷新。设置一个较短的过期时间，可以确保当题目信息有更新时，系统能在几分钟内自动获取最新数据，从而很好地平衡了**性能与一致性**。

1. **分布式缓存 (Redis): 建议 1小时 随机偏移量**

- **理由**: Redis作为二级缓存，是本地缓存未命中时的“坚强后盾”。它的过期时间应该更长一些，以确保当本地缓存失效时，绝大部分请求依然能在Redis层被拦截，而不会穿透到数据库



### 二级缓存下如何保证数据库与缓存的数据一致性

 先操作数据库，再删缓存；本地缓存必须延迟双删，Redis 建议也做延迟双删。  

- 更新 DB
- 删 Redis
- 延迟几百 ms
- 再删一次 Redis
- 删本地缓存 或者  服务集群下 广播通知所有节点删本地缓存

先删除Redis缓存再删除本地缓存：  如果是先删除本地缓存，有可能其他线程读取到Redis的旧值从而导致本地缓存数据不一致；







### 如何更新本地缓存

1. 更新数据库，移除Redis缓存和本地缓存 hotKey提供了`JdHotKeyStore.remove()`方法
2. 查询的时候再使用双重检查锁去读库重建Redis缓存
3. 不过一般情况下，热点信息都是不太会变更的数据，过期时间设置短一点即可。



### 客户端收到worker推送的热Key，但value呢？

**HotKey只告诉“哪个key是热的”，value需要自己手动填充（后面才使用AOP）。框架提供了热key探测能力和本地缓存容器，但里面的数据需要你通过smartSet()放进去。**



### 那在热Key还没有value的时候，大量请求访问该热Key，导致往Redis里去获取值，并批量写入本地缓存。 不会有这种情况吗

会有，框架并没有帮我们去实现 读库到设置值的 并发安全性，需要我们手动实现**双重检查锁**(或者叫互斥锁+重试机制也行)

```java
// 伪代码逻辑
public Object getValue(String key) {
    // 1. 先查本地缓存
    Object value = localCache.get(key);
    if (value != null) return value;
    
    // 2. 同步块/锁：防止大量请求同时进入
    synchronized (key.intern()) {
        // 3. 双重检查：进去后再查一次
        value = localCache.get(key);
        if (value != null) return value;
        
        // 4. 只有第一个请求会执行这一行
        value = loadFromRedisOrDB(key); 
        
        // 5. 把value塞进本地缓存
        localCache.put(key, value);
    }
    return value;
}
```





### Redis VS JVM内存

Redis是单线程结构，上限取决于单核CPU性能，一个标准分片（比如 4GB 内存）的参考 QPS 大约在 **8万到10万** 之间。突然百万级别的请求访问Redis分片集群的一个分片的一个热Key,那该分片由于单线程，请求排队，无法处理这么多请求量。分片会瘫痪。从而影响到其他数据。在极短的窗口内，无法通过快速扩容10倍以上的分片

在4核8G的配置下，Redis单分片实测QPS大约在**10-13万**左右，本地缓存如Caffeine能达到**30-50万以上**



**我本机上使用自带的redis-banchMark 测的QPS在5w左右**



**即使本地缓存挂了，也总比数据库挂了好**