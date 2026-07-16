---
title: "Redis Lua 脚本实战指南"
description: Redis Lua 脚本的核心原理、实战场景与常见问题，涵盖原子性、EVALSHA、分布式锁等知识
date: 2026-06-14T14:59:30+08:00
categories:
    - 脚本
    - 事务
    - 原子性
tags:
    - Redis
    - lua
draft: false
image: 
math: 
license: 
comments: true
build:
    list: always    # Change to "never" to hide the page from the list
---
## 为什么要使用 `Lua`
核心卖点 ：原子性 + 减少网络往返 + 代码复用

- 对比：传统方式：GET + 判断 + SET（多次往返），需要多次网络请求，而`Lua`只需要一次网络调用
- `Lua`方式：一次性完成，保证原子性

----

## 原理

### 原子性

- **ACID 的原子性**：命令要么全执行，要么全部不执行；
- **Redis Lua 脚本的原子性**：整个 Lua 脚本在执行期间，不会被其他客户端的命令打断。至于 Lua 脚本里面的命令是否必须全部成功，或者全部失败，并不要求。

**Redis 如何保证 Lua 脚本的原子性？**

Redis 通过**单线程执行期间暂停一切外部请求**来保证 Lua 脚本在并发层面绝对原子。当 Lua 脚本开始执行时，Redis 会进入一个"脚本执行模式"，在此期间，所有其他客户端的命令都会被阻塞，直到脚本执行完毕或被终止。

### 注意事项

1. **已执行的命令不会回滚**：Lua 脚本中的命令一旦执行，即使后续命令失败，已经执行的命令也不会回滚；
2. **避免耗时操作**：Lua 脚本一定不要包含过于耗时、过于复杂的逻辑，否则会阻塞整个 Redis 实例。

### KEYS 和 ARGV 参数规范

**KEYS** 数组用于传递 Redis 键名：

```lua
-- KEYS[1], KEYS[2], ... 对应传入的键名
redis.call('GET', KEYS[1])
```

**ARGV** 数组用于传递非键名的参数，如过期时间、键值：

```lua
-- ARGV[1], ARGV[2], ... 对应传入的参数
redis.call('SET', KEYS[1], ARGV[1], 'EX', ARGV[2])
```

> **遵循 KEYS 和 ARGV 的分工**：虽然可以将键名传递给 `ARGV`，但不推荐这样做。尤其在 Redis 集群模式下，`KEYS` 的使用有助于定位键所在的节点。

### 集群模式下的限制

在集群模式下，因为集群需要根据键来路由到正确的分片，所以需要明确指定 Lua 脚本要操作的所有 Redis 键，并且**所有键必须位于同一个哈希槽**，否则脚本会执行失败，错误码为 `CROSSSLOT Keys in request don't hash to the same slot`。

----

## 如何使用

### EVAL 和 EVALSHA

执行 Lua 脚本有两种方式：

#### EVAL：直接传递脚本内容

```bash
redis-cli EVAL "return redis.call('GET', KEYS[1])" 1 mykey
```

每次调用都需要传递完整的脚本内容，适合脚本较短或调用不频繁的场景。

#### EVALSHA：通过脚本哈希调用

当脚本较长或需要频繁调用时，建议先将脚本加载到 Redis 内存中，然后通过 SHA1 哈希值调用：

```bash
# 1. 加载脚本，返回 SHA1 哈希值
redis-cli SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# 返回："abc123def456..."

# 2. 通过哈希值调用
redis-cli EVALSHA abc123def456 1 mykey
```

**优势**：减少网络传输的数据量，提升性能。

**注意**：脚本会被缓存在 Redis 内存中，直到重启或执行 `SCRIPT FLUSH` 命令清除。

### 批量操作

对100 个热门商品需要提前写入 `Redis` 缓存（预热）。如果不使用 `Lua`，需要先循环 `EXISTS` 检查，再循环 `SET` 写入，这需要 200 次网络请求，且无法保证原子性。

1. 客户端代码：

```java
public class CachePreheatService {
    private final StringRedisTemplate stringRedisTemplate;
    private DefaultRedisScript<Long> redisScript;
    @PostConstruct
    public void init() {
        try {
            redisScript = new DefaultRedisScript<>();
            redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("lua/batch_preheat.lua")));
            redisScript.setResultType(Long.class);  // 因为返回 0 或 1
            log.info("Lua脚本加载成功");
        } catch (Exception e) {
            log.error("redisScript init lua error", e);
        }
    }
    public boolean batchPreheat(List<String> keys, List<String> values, List<Long> ttls) {
        if (keys == null || values == null || ttls == null) {
            throw new IllegalArgumentException("参数不能为 null");
        }
        if (keys.size() != values.size() || keys.size() != ttls.size()) {
            throw new IllegalArgumentException("keys, values, ttls 大小必须一致");
        }

        // 构建 ARGV 数组：按照 [value1, ttl1, value2, ttl2, ...] 顺序
        Object[] args = new Object[keys.size() * 2];
        for (int i = 0; i < keys.size(); i++) {
            args[i * 2] = values.get(i);
            args[i * 2 + 1] = String.valueOf(ttls.get(i));  // 转为字符串（因为 stringRedisTemplate 序列化字符串）
        }

        try {
            // 执行 Lua 脚本
            Long result = stringRedisTemplate.execute(
                redisScript,
                keys,   // 对应 KEYS
                args    // 对应 ARGV（可变参数，自动展开）
            );
            return result != null && result == 1L;
        } catch (Exception e) {
            log.error("执行批量预热脚本失败", e);
            return false;
        }
    }
}
```

2. lua脚本

```lua
-- batch_preheat.lua
-- KEYS[] : 传入所有需要写入的 Key（如 product:1, product:2, ...）
-- ARGV[] : 按照 [value1, ttl1, value2, ttl2, ...] 的顺序传入

local keys = KEYS
local total = #keys  -- 获取 Key 的总数

-- 1. 【原子性检查】：遍历检查是否任何一个 Key 已存在
for i = 1, total do
    if redis.call('EXISTS', keys[i]) == 1 then
        -- 只要发现一个 Key 存在，立即返回 0 表示失败，且不进行任何修改
        return 0 
    end
end

-- 2. 【批量写入】：全部不存在，开始遍历写入并设置独立的 TTL
-- 注意：ARGV 中索引计算 (i-1)*2 + 1 取值， (i-1)*2 + 2 取 ttl
for i = 1, total do
    local value = ARGV[(i-1) * 2 + 1]
    local ttl = ARGV[(i-1) * 2 + 2]
    -- 执行 SET key value EX ttl
    redis.call('SET', keys[i], value, 'EX', ttl)
end

-- 返回 1 表示全部预热成功
return 1
```

### `Redis` 分布式锁

如果不引入 `Redisson`（其底层大量采用 Lua 脚本实现），直接使用 Redis 实现分布式锁则需要采用 Lua 脚本。

> 手写 `Redis` 的分布式锁时，采用 `Redis` 的原子命令 `SETNX` 去获取锁，而在线程执行完业务逻辑后要删除锁时，不建议直接 `DEL`，因为有一种情况——**锁误删**：当当前线程持锁超时，会自动释放锁，导致其他线程获取到锁，此时当前线程去删除锁，从而导致误删他人锁的情况。通过 `Lua` 脚本实现 `GET` 和 `DEL` 的原子性，从而保证删锁前判断是否当前线程占有。

1. 客户端代码

```java
@Component
public class DistributedLock {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<Long> unlockScript;

    public DistributedLock(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
        // 初始化解锁脚本
        this.unlockScript = new DefaultRedisScript<>();
        this.unlockScript.setScriptText(
            "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('DEL', KEYS[1]) " +
            "else return 0 end"
        );
        this.unlockScript.setResultType(Long.class);
    }

    /**
     * 尝试加锁（非阻塞）
     */
    public String tryLock(String lockKey, long leaseTime, TimeUnit unit) {
        // value = 设置唯一标识= uuid+线程id
        String value = UUID.randomUUID().toString() + Thread.currentThread().getId();
        Boolean success = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, value, leaseTime, unit);
        return Boolean.TRUE.equals(success) ? value : null;
    }

    /**
     * 原子解锁
     */
    public boolean unlock(String lockKey, String lockValue) {
        if (lockValue == null) return false;
        Long result = redisTemplate.execute(
            unlockScript,
            Collections.singletonList(lockKey),
            lockValue
        );
        return result != null && result == 1L;
    }
}
```

```java
@Service
public class OrderService {
    private final DistributedLock lock;
    public void processOrder(String orderId) {
        String lockKey = "order:" + orderId;
        String lockValue = null;
        try {
            // 尝试获取锁，持有 10 秒
            lockValue = lock.tryLock(lockKey, 10, TimeUnit.SECONDS);
            if (lockValue == null) {
                // 获取失败，重试或返回
                return;
            }
            // 执行核心业务...
        } finally {
            // 确保释放锁
            if (lockValue != null) {
                boolean unlocked = lock.unlock(lockKey, lockValue);
                if (!unlocked) {
                    log.warn("解锁失败，可能锁已过期或被其他线程持有");
                }
            }
        }
    }
}
```

2. lua脚本

```lua
-- unlock.lua
-- KEYS[1] : 锁的 key
-- ARGV[1] : 锁的值（唯一标识）

if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

----

## 问题

### Redis的事务是什么

Redis**事务（Transaction）**指的是一组命令的**批量执行机制**，**不支持回滚**。它通过 `MULTI`、`EXEC`、`DISCARD` 和 `WATCH` 四个命令来实现。

使用 Redis 事务的标准流程如下（以命令行为例）：

1. **开启事务 (`MULTI`)**：客户端发送 `MULTI` 命令，Redis 返回 `OK`，此时进入事务模式。
2. **命令入队**：客户端发送后续的多个命令（如 `SET`、`INCR`）。注意，此时**命令并不会立即执行**，而是全部被放入一个队列中。
3. **执行事务 (`EXEC`)**：客户端发送 `EXEC` 命令，Redis 会**依次执行**队列中的所有命令，并将所有结果一次性返回给客户端。

如果事务在执行前，被监视的 Key 被其他客户端修改了怎么办？Redis 提供了 `WATCH` 命令来实现**乐观锁（CAS，即比较并交换）**。

- **用法**：在 `MULTI` 之前执行 `WATCH key`。
- **机制**：如果在执行 `EXEC` 时，被 `WATCH` 的 Key 被其他客户端改动过，则 `EXEC` 会返回 `nil`（空），**整个事务被取消**，不会执行任何命令。

### 既然 Redis事务能保证原子性，为什么还需要 Lua脚本呢？

1. 使用Redis内置的事务功能，每个命令都需要进行网络通信，这会增加服务器的压力。而将命令放入Lua脚本执行，只需要一次通信，既减轻了服务器压力又提高了执行效率。
2. Redis 事务中，事务队列中的所有命令都必须在 EXEC命令执行才会被执行，对于多个命令之间存在依赖关系，比如后面的命令需要依赖上一个命令结果的场景，Redis事务无法满足，因此 Lua 脚本更适合复杂的场景；
3. Redis 事务能做的 Lua能做，Redis事务做不到的 Lua也能做；

**在 `Redisson` 和大多数现代 Redis 应用中，`Lua` 已经彻底替代了事务的位置。**

### Redis 执行 Lua 脚本时，能否保证原子性？

关于 Lua 脚本的原子性定义及单线程保证机制，详见「原理」章节。这里重点说明不同部署方式下的差异：

1. **单机部署**：不管 Lua 脚本中操作的 key 是不是同一个，都能保证原子性；

2. **主从部署**：所有写操作都在主节点上执行，因此能保证原子性。但需要注意：从节点异步复制主节点的数据，可能存在延迟。如果在从节点上执行脚本中的读操作，可能会读到稍有延迟的数据；

3. **Cluster 部署**：
   - **如果 Key 被哈希到不同的 Slot**：**直接报错拒绝执行**，错误码是 `CROSSSLOT Keys in request don't hash to the same slot`；
   - **如果 Key 恰好被哈希到相同的 Slot**：它们就在同一个节点上，此时能保证原子性。

### `redis.call()` 和 `redis.pcall()`

`redis.call()` 用于执行 Redis 的命令。当命令执行出错时，会阻断整个脚本执行，并将错误信息返回给客户端。

`redis.pcall()` 也用于执行 Redis 的命令。当命令执行出错时，不会阻断脚本的执行，而是内部捕获错误，并继续执行后续的命令。

### 脚本死循环怎么办？

`SCRIPT KILL` 用于动态杀死一个执行时间超时的 Lua 脚本。不过 `SCRIPT KILL` 的执行有一个重要的前提，那就是当前正在执行的脚本**没有对 Redis 的内部数据状态进行修改**，因为 Redis 不允许 `SCRIPT KILL` 破坏脚本执行的原子性。比如脚本内部使用了 `redis.call("SET", key, value)` 修改了内部的数据，那么 `SCRIPT KILL` 执行时服务器会返回错误。

如果脚本已经对数据进行了修改，此时唯一的解决方案是执行 `SHUTDOWN NOSAVE`：

`SHUTDOWN NOSAVE` 会强制关闭 Redis 服务，并且**不会持久化最近的数据变更**。这意味着脚本执行期间的所有修改都会丢失，但可以立即解除阻塞状态。

----

## 参考文档

[利用Redis的SetNx一步步实现分布式锁并改进_redis 分布式锁 setnx lua-CSDN博客](https://blog.csdn.net/Drifter_Galaxy/article/details/130555091)

[java中采用lua脚本执行redis操作_java使用lua脚本操作 redis-CSDN博客](https://blog.csdn.net/xiaozhu0301/article/details/111469101)

[(8 封私信 / 80 条消息) 阿里 P7二面：Redis 执行 Lua，能保证原子性吗？ - 知乎](https://zhuanlan.zhihu.com/p/684155151)

[(8 封私信 / 80 条消息) 欲求不满之 Redis Lua脚本的执行原理 - 知乎](https://zhuanlan.zhihu.com/p/48337244)