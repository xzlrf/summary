# Redisson 分布式锁：从原理到实战

> 一次把分布式锁的所有坑踩完

## 目录

- [一、为什么需要分布式锁](#一为什么需要分布式锁)
- [二、Redis 分布式锁的演进之路](#二redis-分布式锁的演进之路)
  - [2.1 第一代：SETNX + 手动过期](#21-第一代setnx--手动过期)
  - [2.2 第二代：SET NX EX 原子命令](#22-第二代set-nx-ex-原子命令)
  - [2.3 第三代：Lua 脚本保障原子性](#23-第三代lua-脚本保障原子性)
  - [2.4 第四代：Redisson 完整框架](#24-第四代redisson-完整框架)
- [三、Redisson 核心原理](#三redisson-核心原理)
  - [3.1 底层数据结构](#31-底层数据结构)
  - [3.2 加锁流程](#32-加锁流程)
  - [3.3 释放锁流程](#33-释放锁流程)
- [四、Redisson 解决了哪些问题](#四redisson-解决了哪些问题)
  - [4.1 死锁问题：过期时间兜底](#41-死锁问题过期时间兜底)
  - [4.2 锁误删问题：Lua 原子性保障](#42-锁误删问题lua-原子性保障)
  - [4.3 可重入性：Hash 计数器](#43-可重入性hash-计数器)
  - [4.4 看门狗机制：自动续期](#44-看门狗机制自动续期)
- [五、Redisson 进阶知识点](#五redisson-进阶知识点)
  - [5.1 公平锁实现](#51-公平锁实现)
  - [5.2 红锁 RedLock](#52-红锁-redlock)
  - [5.3 读写锁](#53-读写锁)
  - [5.4 联锁 MultiLock](#54-联锁-multilock)
  - [5.5 信号量 Semaphore](#55-信号量-semaphore)
- [六、生产环境避坑指南](#六生产环境避坑指南)
- [七、总结](#七总结)

---

## 一、为什么需要分布式锁

单机环境下，用 `synchronized` 或 `ReentrantLock` 就能解决线程安全问题。但在分布式系统中，多个 JVM 进程同时访问共享资源（如数据库同一条记录、库存扣减等），Java 内置锁就不够用了——它只在单个 JVM 内有效。

**分布式锁需要满足以下几个核心条件：**

1. **互斥性**：同一时刻只有一个客户端能持有锁
2. **防死锁**：即使持有锁的客户端崩溃，锁也能被释放
3. **锁的归属**：谁加的锁，只能由谁来释放
4. **可重入**：同一个客户端可以对同一把锁多次加锁
5. **高可用**：锁服务本身不能成为单点

Redis 凭借单线程模型 + SETNX 原子操作，天然适合做分布式锁。但裸用 Redis 命令很容易踩坑，Redisson 把这些坑全部填平了。

---

## 二、Redis 分布式锁的演进之路

### 2.1 第一代：SETNX + 手动过期

最早的做法是用 `SETNX`（SET if Not eXists）命令：

```java
// 加锁
Boolean locked = jedis.setnx("lock:order:1001", "thread-1");
if (locked) {
    try {
        // 业务逻辑
    } finally {
        // 释放锁
        jedis.del("lock:order:1001");
    }
}
```

**致命缺陷：** 如果业务逻辑抛异常或者进程崩溃，`finally` 块没执行，锁永远不会被释放 → **死锁**。

### 2.2 第二代：SET NX EX 原子命令

给锁加上过期时间：

```java
// Redis 2.6.12+ 支持 SET + NX + EX 原子操作
jedis.set("lock:order:1001", "thread-1", "NX", "EX", 30);
```

这一步解决了死锁问题：即使客户端崩溃，30 秒后锁自动过期释放。

**新问题：** 业务执行时间如果超过 30 秒，锁自动释放了，其他线程就能拿到锁，导致并发安全问题。而且锁被其他线程持有后，原线程执行完还会误删别人的锁。

### 2.3 第三代：Lua 脚本保障原子性

释放锁时必须保证"判断归属 + 删除"是原子操作：

```lua
-- 释放锁的 Lua 脚本
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

Java 端调用：

```java
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) else return 0 end";
jedis.eval(script, Collections.singletonList("lock:order:1001"),
           Collections.singletonList("thread-1"));
```

这样就不会出现"先判断再删除"之间被其他线程插入的问题。

### 2.4 第四代：Redisson 完整框架

Redisson 在上述基础上做了完整封装：
- 可重入锁（同一个线程多次加锁不会死锁自己）
- 看门狗自动续期（不用担心业务超时锁被释放）
- 公平锁 / 红锁 / 读写锁 / 联锁等丰富变体
- 基于 Redis 发布订阅的等待线程唤醒机制

下面深入看 Redisson 是怎么做的。

---

## 三、Redisson 核心原理

### 3.1 底层数据结构

Redisson 可重入锁在 Redis 中用 **Hash** 结构存储：

```
KEY: "redisson_lock:myLock"
VALUE (Hash):
  {
    "uuid:threadId": 1   // value 是重入计数
  }
```

- **Key** 是锁的名称（带前缀 `redisson_lock:`）
- **Hash 的 field** 是 `UUID:线程ID`，唯一标识哪个客户端的哪个线程持有了锁
- **Hash 的 value** 是重入计数器，第一次加锁值为 `1`，重入一次加 `1`，释放时减 `1`

**为什么用 Hash 而不是 String？**

1. String 只能存一个值，无法区分哪个线程持有锁
2. Hash 天然支持 field 级别的原子增减，方便实现可重入
3. 多个线程等待同一把锁时，Hash 可以扩展为排队队列

### 3.2 加锁流程

Redisson 加锁的核心逻辑简化后如下（Lua 脚本）：

```lua
-- 尝试加锁
if (redis.call('exists', KEYS[1]) == 0) then
    -- 锁不存在，可以加锁
    redis.call('hincrby', KEYS[1], ARGV[2], 1);  -- 重入计数+1
    redis.call('expire', KEYS[1], ARGV[1]);       -- 设置过期时间
    return nil;                                    -- 返回 nil 表示加锁成功
end;

-- 锁已存在，判断是否是当前线程（可重入）
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);  -- 重入计数+1
    redis.call('expire', KEYS[1], ARGV[1]);       -- 刷新过期时间
    return nil;                                    -- 可重入成功
end;

-- 锁被其他线程持有，返回剩余 TTL
return redis.call('pttl', KEYS[1]);
```

加锁流程图：

```
加锁请求
  │
  ├─ 锁不存在 → 创建 Hash field，计数=1，设过期时间 → 成功返回
  │
  ├─ 锁存在，且 field 匹配 → 计数+1，刷新过期时间 → 成功返回（可重入）
  │
  └─ 锁存在，且 field 不匹配 → 返回剩余 TTL → 客户端订阅 Pub/Sub → 阻塞等待
```

等待线程通过 Redis 发布订阅机制监听锁释放事件，而不是轮询。这比轮询高效得多。

### 3.3 释放锁流程

释放锁的 Lua 脚本：

```lua
-- 判断当前线程是否持有锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
    return nil;   -- 不是当前线程持有的锁，不操作
end;

-- 重入计数 -1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);

-- 计数 > 0，说明还在重入范围内，只刷新过期时间
if (counter > 0) then
    redis.call('expire', KEYS[1], ARGV[2]);
    return 0;
-- 计数 = 0，彻底释放锁，删除 key，并发布解锁消息
else
    redis.call('del', KEYS[1]);
    redis.call('publish', KEYS[2], ARGV[1]);  -- 唤醒等待线程
    return 1;
end;
return nil;
```

关键点：
1. 先判断归属，不是自己的锁不操作（解决锁误删）
2. 可重入计数 -1，没归零就不真正删除（支持可重入）
3. 真正释放后通过 `publish` 唤醒等待线程

---

## 四、Redisson 解决了哪些问题

### 4.1 死锁问题：过期时间兜底

**问题描述**：客户端加锁后崩溃，锁永远不会被释放，其他所有线程永久阻塞。

**Redisson 的方案**：每次加锁都会设置过期时间。默认是 `lockWatchdogTimeout`（30 秒）。

```java
// Redisson 源码常量
private static final long LOCK_EXPIRATION_INTERVAL = 30_000L; // 30秒
```

即使客户端进程直接 `kill -9`，最多 30 秒后锁自动释放。

但光有过期时间还不够——如果业务执行超过 30 秒怎么办？这就引出了看门狗机制。

### 4.2 锁误删问题：Lua 原子性保障

**问题场景**：

```
线程 A 加锁，设置过期 10 秒
线程 A 业务执行很慢，10 秒后锁自动过期释放
线程 B 拿到锁，开始执行业务
线程 A 终于执行完，执行 del 命令 → 把线程 B 的锁删了！
线程 C 也拿到锁... 并发灾难
```

**Redisson 的方案**：释放锁时，用 Lua 脚本保证"判断归属 + 删除"原子执行：

```lua
if redis.call('hexists', KEYS[1], ARGV[3]) == 0 then
    return nil  -- 不是自己的锁，直接返回，不删
end
```

因为 Lua 脚本在 Redis 中是原子执行的，不存在"判断完、删除前"被其他线程插入的可能。

**但要注意**：Redisson 的 Lua 方案解决的是**主动释放时的误删**。如果业务超时导致锁被看门狗放弃续期、自动过期后误删，Redisson 本身无法阻止。这是所有基于 Redis 的分布式锁都无法彻底解决的问题，只能靠合理设置超时时间来规避。

### 4.3 可重入性：Hash 计数器

**什么是可重入**：同一个线程对同一把锁多次加锁，不会把自己阻塞住。

Java 的 `ReentrantLock` 就是可重入的，Redisson 也实现了这一点。

**实现原理**：

```
锁不存在:
  HSET lockName "uuid:threadId" 1
  EXPIRE lockName 30

同一个线程再次加锁（可重入）:
  HINCRBY lockName "uuid:threadId" 1  → 值变为 2
  EXPIRE lockName 30

第一次释放:
  HINCRBY lockName "uuid:threadId" -1  → 值变为 1
  EXPIRE lockName 30  （还没归零，只刷新过期时间）

第二次释放:
  HINCRBY lockName "uuid:threadId" -1  → 值变为 0
  DEL lockName  （归零，真正删除）
```

为什么需要可重入？看一个典型场景：

```java
RLock lock = redisson.getLock("inventory");

public void methodA() {
    lock.lock();
    try {
        methodB();  // methodB 也会尝试加同一把锁
    } finally {
        lock.unlock();
    }
}

public void methodB() {
    lock.lock();  // 如果不可重入，这里就把自己阻塞了
    try {
        // 扣减库存
    } finally {
        lock.unlock();
    }
}
```

### 4.4 看门狗机制：自动续期

**问题**：业务执行时间不确定。设太短的过期时间，锁可能在业务执行中自动释放；设太长，客户端崩溃后其他线程要等很久。

**Redisson 的方案**：看门狗（WatchDog）——后台线程定时续期。

**工作流程**：

```
加锁成功（默认 30 秒过期）
  │
  ▼
启动后台线程（看门狗）
  │
  ▼
每隔 10 秒（30 / 3）检查一次
  ├─ 当前线程还持有锁 → EXPIRE 刷新到 30 秒
  └─ 当前线程已释放锁 → 停止续期，线程退出
```

看门狗核心代码逻辑（简化版）：

```java
// Redisson 源码中的续期调度
private void scheduleExpirationRenewal(long threadId) {
    renewalTask = new TimeoutTask();
    renewalTask = commandExecutor.getConnectionManager()
        .newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) {
                // Lua 脚本续期：刷新过期时间为 30 秒
                RFuture<Boolean> future = commandExecutor.evalWriteAsync(
                    getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                    "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                    "    redis.call('expire', KEYS[1], ARGV[1]); " +
                    "    return 1; " +
                    "end; " +
                    "return 0;",
                    Collections.singletonList(getName()),
                    internalLockLeaseTime, getLockName(threadId)
                );

                future.onResponse((res, e) -> {
                    if (res && !e) {
                        // 续期成功，10 秒后再执行
                        scheduleExpirationRenewal(threadId);
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
}
```

**关键参数**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `lockWatchdogTimeout` | 30 秒 | 锁的默认过期时间 |
| 续期间隔 | 10 秒 | `lockWatchdogTimeout / 3` |
| 续期条件 | 当前线程仍持有锁 | 通过 `HEXISTS` 判断 |

**看门狗的局限**：

1. 只有调用 `lock()`（不传过期时间）时才启动看门狗
2. 调用 `lock(leaseTime, TimeUnit)` 指定过期时间时，看门狗**不启动**
3. 如果 Redis 主从切换导致锁丢失，看门狗也无力回天（这是 RedLock 解决的问题）

**实际使用建议**：

```java
// ✅ 推荐：使用看门狗，不传过期时间
lock.lock();

// ❌ 指定过期时间，看门狗不启动，超时后锁自动释放
lock.lock(10, TimeUnit.SECONDS);
```

---

## 五、Redisson 进阶知识点

### 5.1 公平锁实现

Redisson 的公平锁基于 Redis 的 **List + Pub/Sub** 实现排队：

```java
RLock fairLock = redisson.getFairLock("myLock");
fairLock.lock();
```

**原理**：

1. 客户端尝试加锁时，将自己的 `clientId` 追加到 List 尾部：`RPUSH lockName:queue clientId`
2. 检查自己是否是 List 的第一个元素
   - 是 → 加锁成功
   - 不是 → 订阅该锁的 Pub/Sub 频道，阻塞等待
3. 释放锁时，弹出 List 第一个元素，并通过 Pub/Sub 通知下一个等待者

相比非公平锁（直接竞争，先到先得无保证），公平锁按请求顺序获取锁，适合对顺序敏感的场景。

### 5.2 红锁 RedLock

RedLock 是 Redis 作者 Antirez 提出的多 Redis 实例分布式锁算法：

```java
RLock lock1 = redisson1.getLock("myLock");
RLock lock2 = redisson2.getLock("myLock");
RLock lock3 = redisson3.getLock("myLock");

RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
redLock.lock();
```

**算法核心**：

1. 依次向 N 个独立的 Redis 实例加锁（5 个）
2. 计算加锁总耗时
3. 如果在 **过半** 实例上加锁成功，且总耗时 < 锁有效时间，则加锁成功
4. 否则向所有实例释放锁，重新尝试

**争议**：Martin Kleppmann 曾发论文质疑 RedLock 的正确性（时钟跳跃、GC 停顿等场景下仍可能出问题）。但在大多数生产环境中，RedLock 已经比单实例方案安全得多。

### 5.3 读写锁

Redisson 支持读写锁（ReadWriteLock），读读不互斥，读写/写写互斥：

```java
RReadWriteLock rwLock = redisson.getReadWriteLock("myLock");
RLock readLock = rwLock.readLock();
RLock writeLock = rwLock.writeLock();

readLock.lock();   // 多个读线程可同时持有
writeLock.lock();  // 写线程独占
```

**底层实现**：

- 读锁：Redis Hash 中记录多个 reader 的 clientId，只要存在 reader，写锁就无法获取
- 写锁：独占，有 reader 或 writer 存在时都不能获取

### 5.4 联锁 MultiLock

MultiLock 要求**所有**锁实例都加锁成功才算成功（与 RedLock 的"过半"不同）：

```java
RLock lock1 = redisson1.getLock("myLock");
RLock lock2 = redisson2.getLock("myLock");

RedissonMultiLock multiLock = new RedissonMultiLock(lock1, lock2);
multiLock.lock();  // lock1 和 lock2 都必须成功
```

### 5.5 信号量 Semaphore

Redisson 基于 Redis 实现了信号量：

```java
RSemaphore semaphore = redisson.getSemaphore("mySemaphore");
semaphore.trySetPermits(5);  // 设置 5 个许可

semaphore.acquire();  // 获取一个许可
// 业务逻辑...
semaphore.release();  // 释放一个许可
```

底层用 Redis Hash 存储许可数量，通过 Lua 脚本原子增减。

---

## 六、生产环境避坑指南

### 坑 1：指定过期时间导致看门狗不启动

```java
// ❌ 看门狗不启动，业务超时后锁自动释放
lock.lock(5, TimeUnit.SECONDS);

// ✅ 让看门狗管理
lock.lock();
```

### 坑 2：Redis 主从切换导致锁丢失

单实例 Redis 的 AOF 默认是 `everysec`，最多丢 1 秒数据。主从异步复制，主节点加锁后如果还没同步到从节点就宕机，新主节点上就没有这把锁。

**解决**：使用 RedLock 部署多实例，或接受 Redis 分布式锁的 CAP 取舍（Redis 选 AP，不是强一致）。

### 坑 3：锁粒度太粗

```java
// ❌ 锁住整个库存
RLock lock = redisson.getLock("inventory");

// ✅ 锁住具体商品
RLock lock = redisson.getLock("inventory:product:1001");
```

### 坑 4：忘记释放锁或异常路径没释放

```java
// ❌ 业务异常时锁不释放
lock.lock();
doSomething();   // 抛异常 → finally 没执行
lock.unlock();

// ✅ 标准写法
lock.lock();
try {
    doSomething();
} finally {
    // 只有当前线程持有锁才释放
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 坑 5：大量线程竞争同一把锁导致 Redis 压力

每个加锁请求都是一次 Redis 命令调用，高并发场景下可能成为瓶颈。

**建议**：
- 锁的 key 尽量分散（按业务维度拆分）
- 业务执行时间尽量短，减少锁持有时间
- 如果并发极高，考虑改用分段锁或其他并发方案

### 坑 6：Redisson 版本 bug

老版本 Redisson 存在续期任务内存泄漏、Pub/Sub 断连等问题。建议 **3.17.0+**。

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.27.0</version>
</dependency>
```

---

## 七、总结

Redisson 分布式锁的核心知识点可以归纳为四句话：

| 问题 | 解决方案 | 实现手段 |
|------|----------|----------|
| 死锁 | 过期时间兜底 | `EXPIRE` 命令 |
| 锁误删 | 判断归属再删除 | Lua 脚本原子执行 |
| 可重入 | 同一线程多次加锁 | Hash 计数器 + `HINCRBY` |
| 业务超时锁被释放 | 自动续期 | 看门狗后台线程，每 10 秒续期一次 |

**设计取舍**：Redis 分布式锁本质是 AP 模型（可用 + 分区容忍），不是强一致的。在大多数业务场景（库存扣减、防止重复提交、定时任务互斥等）完全够用。如果你的场景要求强一致（如金融级），需要考虑 ZooKeeper 或 etcd 实现的分布式锁（CP 模型）。

**一句话总结**：Redisson 在 Redis 原生命令的基础上，用 Hash 结构实现了可重入，用 Lua 脚本保障了原子性，用后台线程实现了看门狗续期，用 Pub/Sub 实现了高效等待——这些加起来才是一个生产可用的分布式锁。
