# 缓存一致性：Redis 缓存与数据库的相爱相杀

> 搞懂缓存一致性，面试和工作都够用

## 目录

- [一、为什么会有缓存一致性问题](#一为什么会有缓存一致性问题)
- [二、经典方案：先更新 DB 再删除缓存](#二经典方案先更新-db-再删除缓存)
  - [2.1 为什么不是"先删缓存再更 DB"](#21-为什么不是先删缓存再更-db)
  - [2.2 为什么是"删缓存"而不是"更新缓存"](#22-为什么是删缓存而不是更新缓存)
  - [2.3 并发场景下的完整推演](#23-并发场景下的完整推演)
  - [2.4 这个方案还有没有问题](#24-这个方案还有没有问题)
- [三、进阶方案：延迟双删](#三进阶方案延迟双删)
  - [3.1 什么是延迟双删](#31-什么是延迟双删)
  - [3.2 延迟时间怎么算](#32-延迟时间怎么算)
  - [3.3 延迟双删的完整流程](#33-延迟双删的完整流程)
  - [3.4 延迟双删的局限性](#34-延迟双删的局限性)
- [四、最终一致性：异步淘汰方案](#四最终一致性异步淘汰方案)
  - [4.1 基于 Binlog 的订阅删除](#41-基于-binlog-的订阅删除)
  - [4.2 Canal + MQ 架构](#42-canal--mq-架构)
  - [4.3 重试与补偿机制](#43-重试与补偿机制)
  - [4.4 兜底策略：缓存过期时间](#44-兜底策略缓存过期时间)
- [五、读场景的缓存一致性](#五读场景的缓存一致性)
  - [5.1 缓存穿透](#51-缓存穿透)
  - [5.2 缓存击穿](#52-缓存击穿)
  - [5.3 缓存雪崩](#53-缓存雪崩)
- [六、方案对比与选型建议](#六方案对比与选型建议)
- [七、总结](#七总结)

---

## 一、为什么会有缓存一致性问题

典型的读写架构是这样的：

```
  写请求                    读请求
    │                         │
    ▼                         ▼
┌─────────┐            ┌──────────┐
│  MySQL  │            │  Redis   │
│  (DB)   │ ◄──同步──► │  (Cache) │
└─────────┘            └──────────┘
```

**问题**：当数据被修改后，DB 和 Cache 中的数据会出现短暂（或长期）不一致的窗口。

例如：用户修改了商品库存，DB 已经更新为 99，但 Redis 里还是 100。此时另一个用户来读，拿到了脏数据。

缓存一致性的核心命题就是：**如何在性能和一致性之间找到可接受的平衡。**

---

## 二、经典方案：先更新 DB 再删除缓存

这是业界最广泛采用的方案，也是面试的标准答案。

### 2.1 为什么不是"先删缓存再更 DB"

反推一个场景就能明白：

```
线程 A（写操作）          线程 B（读操作）
    │                        │
①  删除缓存 ✓               │
    │                        │
②  还没更新 DB...            │
    │                     ③  读缓存 → 不命中
    │                     ④  读 DB → 旧值 100
    │                     ⑤  写回缓存 → 旧值 100
    │                        │
⑥  更新 DB → 新值 99        │
    │                        │
    │         结果：DB=99, 缓存=100  ❌ 不一致！
```

先删缓存再更 DB，在并发读写的情况下会导致缓存里一直是旧数据。

### 2.2 为什么是"删缓存"而不是"更新缓存"

有人可能会问：既然要维护一致性，为什么不直接把缓存也更新成新值？

**两个原因：**

1. **性能**：删除是 O(1) 操作，更新缓存需要重新计算/查询完整数据。如果缓存值是多个 DB 字段组合的结果（如聚合数据），更新缓存的代价远大于删除。

2. **正确性**：写操作往往只改一个字段，但缓存可能包含多个字段。如果只更新缓存中对应的字段，需要知道完整的缓存结构；而删除后让下一次读取重新加载，能自动拿到最新全量数据。

```java
// ❌ 更新缓存：需要知道完整的缓存数据结构
String newValue = "只改了库存";
redis.set("product:1001", complexJsonWithAllFields);  // 需要重组整个对象

// ✅ 删除缓存：简单直接
redis.del("product:1001");
// 下次读取时自动从 DB 加载最新数据
```

### 2.3 并发场景下的完整推演

**场景一：两个写操作并发**

```
线程 A（写）                线程 B（写）
    │                          │
①  更新 DB → 库存 = 99        │
    │                          │
②  删除缓存 ✓                 │
    │                     ③   更新 DB → 库存 = 98
    │                     ④   删除缓存 ✓（幂等，删了还是删了）
    │                          │
⑤  读请求 → 缓存不命中         │
⑥  读 DB → 98（最新值）✓      │
⑦  写回缓存 → 98 ✓            │
```

**结论**：最终缓存读到的是最新值，正确。

**场景二：读操作和写操作并发（先更 DB 再删缓存）**

```
线程 A（写）                线程 B（读）
    │                          │
①  更新 DB → 库存 = 99        │
    │                     ②   读缓存 → 命中，旧值 100
    │                          │   （读到了旧数据，但只是一瞬间）
③  删除缓存 ✓                 │
    │                          │
④  下一个读请求 → 不命中       │
⑤  读 DB → 99 ✓              │
```

**结论**：读可能短暂读到旧缓存，但删除操作完成后，下一个读就会拿到新值。不一致窗口 = 删除缓存到读请求之间的时间，通常很短（毫秒级）。

### 2.4 这个方案还有没有问题

**有。** 在极端情况下：

```
线程 A（写）                线程 B（读）
    │                          │
①  更新 DB → 99               │
    │                     ②   读缓存 → 不命中
    │                     ③   读 DB → 99 ✓
    │                     ④   还没写回缓存...
⑤  删除缓存 ✓（删了个空，无影响）│
    │                     ⑥   写回缓存 → 99 ✓
    │                          │
    │         结果一致，看起来没问题
```

正常情况没问题。但如果线程 B 的"写回缓存"步骤延迟了，可能引发另一种场景：

```
线程 A（写1）               线程 C（写2）
    │                          │
①  更新 DB → 99               │
    │                     ②   更新 DB → 98
    │                     ③   删除缓存 ✓
    │                          │
④  删除缓存 ✓                 │
    │                          │
⑤  读请求 → 不命中             │
⑥  读 DB → 98 ✓              │
    │                          │
    │  结果正确
```

**结论**：先更 DB 再删缓存在绝大多数场景下是正确的。唯一的隐患是"删除缓存失败"（Redis 宕机、网络超时），这会导致不一致。需要兜底方案——这就是后面两个方案要解决的问题。

---

## 三、进阶方案：延迟双删

### 3.1 什么是延迟双删

延迟双删 = **先删缓存 → 更新 DB → 休眠一段时间 → 再删缓存**。

它本质上是对"先删缓存再更 DB"方案的补救，通过第二次删除来清除在第一次删除和 DB 更新之间被读请求写回的脏数据。

### 3.2 延迟时间怎么算

延迟时间需要 ≥ 读请求的完整耗时：

```
延迟时间 = 读 DB 时间 + 写缓存时间 + 缓冲余量
```

通常设为 **500ms ~ 1000ms**。经验公式：

```java
int sleepTime = (readDbTimeMs + writeCacheTimeMs) * 2;
// 一般 500ms 就够用了
```

### 3.3 延迟双删的完整流程

```java
public void updateProduct(Long id, ProductDTO dto) {
    // 第 1 次删除：防止脏缓存
    redis.del("product:" + id);

    // 更新数据库
    productMapper.updateById(id, dto);

    // 休眠一段时间（等潜在的读请求完成）
    try {
        Thread.sleep(500);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }

    // 第 2 次删除：清除读请求写回的脏数据
    redis.del("product:" + id);
}
```

**完整时序推演：**

```
线程 A（写）                    线程 B（读）
    │                              │
①  删除缓存 ✓                     │
    │                         ②   读缓存 → 不命中
    │                         ③   读 DB → 旧值 100
    │                         ④   写回缓存 → 旧值 100
⑤  更新 DB → 99                  │
    │                              │
⑥  sleep(500ms)                  │
    │                              │
⑦  第 2 次删除 ✓ → 脏缓存 100 被清掉
    │                              │
⑧  下一个读请求 → 不命中           │
⑨  读 DB → 99 ✓                  │
```

延迟双删保证了：即使第一次删除后有读请求写回了脏缓存，第二次删除也会把它清掉。

### 3.4 延迟双删的局限性

1. **吞吐量下降**：每次写操作都要 sleep，线程阻塞，QPS 直接打折。
2. **延迟时间不好拿捏**：设太短，读请求还没写完就被第二次删除漏掉；设太长，写请求响应慢。
3. **第二次删除可能失败**：和方案一一样，Redis 故障时删除不了，仍然不一致。
4. **代码侵入性强**：业务代码要写 sleep 逻辑，不好维护。

**结论**：延迟双删是一种**妥协方案**，适合对一致性要求较高但又不想引入复杂中间件的场景。生产环境更推荐使用后面的异步淘汰方案。

---

## 四、最终一致性：异步淘汰方案

这是目前生产环境最推荐的方案：**不依赖业务代码，通过订阅数据库变更来异步删除缓存。**

### 4.1 基于 Binlog 的订阅删除

**核心思路**：业务只负责更新 DB，不操作缓存。由一个独立的组件监听 MySQL 的 binlog，当发现有 UPDATE/DELETE 时，自动去删除 Redis 中对应的缓存。

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│  业务层  │ ──▶  │  MySQL   │ ──▶  │  binlog  │ ──▶  │  监听器   │
│          │      │  (更新DB) │      │  (变更流) │      │          │
└─────────┘      └──────────┘      └──────────┘      └────┬─────┘
                                                          │
                                                          ▼
                                                   ┌──────────┐
                                                   │  Redis   │
                                                   │  (删缓存) │
                                                   └──────────┘
```

**优势：**
- 业务代码无侵入：写 DB 就是写 DB，不用管缓存
- 解耦：缓存淘汰逻辑独立运行
- 可靠：即使 Redis 宕机，binlog 还在，恢复后继续消费

### 4.2 Canal + MQ 架构

生产环境常用 **Canal + 消息队列** 来实现：

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│  业务层  │ ──▶  │  MySQL   │ ──▶  │  Canal   │ ──▶  │   MQ     │
│          │      │  (binlog) │      │  (解析)  │      │  (Kafka/ │
└─────────┘      └──────────┘      └──────────┘      │  RabbitMQ)│
                                                     └────┬─────┘
                                                          │
                                                          ▼
                                                   ┌──────────────┐
                                                   │  缓存淘汰服务  │
                                                   │  (消费 MQ    │
                                                   │   删 Redis)  │
                                                   └──────────────┘
```

**各组件职责：**

| 组件 | 职责 |
|------|------|
| MySQL | 业务数据持久化，产生 binlog |
| Canal | 伪装成 MySQL Slave，实时拉取 binlog 并解析 |
| MQ | 缓冲 binlog 事件，保证不丢，支持重试 |
| 缓存淘汰服务 | 消费 MQ 消息，解析表名和主键，删除对应 Redis 缓存 |

**Canal 配置示例：**

```java
// Canal 客户端监听 binlog 变更
public class CanalCacheEvictor {
    public void listen() {
        CanalConnector connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("127.0.0.1", 11111),
            "destination", "", "");

        while (true) {
            connector.connect();
            connector.subscribe(".*\\..*");  // 监控所有表
            Message message = connector.get(1000);
            List<Entry> entries = message.getEntries();

            for (Entry entry : entries) {
                if (entry.getEntryType() == EntryType.ROWDATA) {
                    RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
                    EventType eventType = rowChange.getEventType();

                    if (eventType == EventType.UPDATE || eventType == EventType.DELETE) {
                        // 解析出表名和主键
                        String table = entry.getHeader().getTableName();
                        List<RowData> rowDatasList = rowChange.getRowDatasList();
                        for (RowData rowData : rowDatasList) {
                            // 获取更新前的主键值（beforeColumns）
                            for (Column column : rowData.getBeforeColumnsList()) {
                                if (column.getIsKey()) {
                                    // 删除缓存
                                    String cacheKey = table + ":" + column.getValue();
                                    redis.del(cacheKey);
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### 4.3 重试与补偿机制

异步方案必须考虑"删除失败怎么办"的问题。

**重试策略：**

```java
// 消费 MQ 消息时，删除失败就重试
@RabbitListener(queues = "cache-evict-queue")
public void handleEvict(CacheEvictMessage msg) {
    int retryCount = 0;
    int maxRetries = 3;

    while (retryCount < maxRetries) {
        try {
            redis.del(msg.getCacheKey());
            return;  // 删除成功，ACK
        } catch (Exception e) {
            retryCount++;
            if (retryCount >= maxRetries) {
                // 超过最大重试次数，发送到死信队列
                deadLetterQueue.send(msg);
                log.error("缓存删除失败，已发送到死信队列: {}", msg.getCacheKey());
                return;
            }
            // 指数退避重试
            Thread.sleep(1000 * (long) Math.pow(2, retryCount));
        }
    }
}
```

**补偿任务：**

即使有重试，也可能漏掉。可以加一个定时任务做兜底补偿：

```java
// 定时任务：定期比对 DB 和缓存的数据
@Scheduled(cron = "0 */5 * * * ?")  // 每 5 分钟执行一次
public void compensate() {
    // 1. 查出最近 5 分钟内更新过的数据
    List<Product> updated = productMapper.selectRecentlyUpdated(5);

    for (Product p : updated) {
        String cacheKey = "product:" + p.getId();
        String cacheValue = redis.get(cacheKey);

        // 2. 比对缓存和 DB
        if (cacheValue != null) {
            Product cached = JSON.parseObject(cacheValue, Product.class);
            if (!Objects.equals(cached.getStock(), p.getStock())) {
                // 3. 不一致，删除缓存（让下次读重新加载）
                redis.del(cacheKey);
                log.warn("发现不一致，已删除缓存: {}", cacheKey);
            }
        }
    }
}
```

### 4.4 兜底策略：缓存过期时间

无论用哪种方案，**给缓存设置过期时间**是最基本的兜底：

```java
// 设置 30 分钟过期
redis.setex("product:" + id, 1800, jsonData);
```

这样即使所有淘汰机制都失效，缓存最多不一致的时间也不会超过过期时间。这是最后一道安全网。

---

## 五、读场景的缓存一致性

除了写场景，读操作也有三个经典问题：

### 5.1 缓存穿透

**问题**：请求一个不存在的数据（如 id = -1），缓存不命中，每次都打到 DB。恶意攻击时 DB 直接被打挂。

**解决方案：**

1. **缓存空值**：DB 查到没有数据时，缓存一个特殊值（如 `null` 或空对象），设置较短过期时间：

```java
Product product = productMapper.selectById(id);
if (product == null) {
    redis.setex("product:" + id, 60, "NULL");  // 缓存空值，60 秒过期
    return null;
}
```

2. **布隆过滤器**：在缓存层之前加一层 Bloom Filter，快速判断数据是否存在：

```java
// 启动时把所有有效 ID 加载到布隆过滤器
BloomFilter<Long> bloomFilter = BloomFilter.create(
    Funnels.longFunnel(), expectedInsertions, falsePositiveRate);

// 读请求先过布隆过滤器
if (!bloomFilter.mightContain(id)) {
    return null;  // 肯定不存在，直接返回
}
```

### 5.2 缓存击穿

**问题**：某个热点 key 过期瞬间，大量请求同时到达 DB。

**解决方案：**

1. **互斥锁（分布式锁）**：只让一个线程去 DB 加载，其他线程等待：

```java
public Product getProduct(Long id) {
    String cacheKey = "product:" + id;
    String value = redis.get(cacheKey);
    if (value != null) {
        return JSON.parseObject(value, Product.class);
    }

    // 缓存不命中，尝试获取分布式锁
    RLock lock = redisson.getLock("lock:product:" + id);
    try {
        if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
            // 双重检查
            value = redis.get(cacheKey);
            if (value != null) {
                return JSON.parseObject(value, Product.class);
            }
            // 从 DB 加载
            Product product = productMapper.selectById(id);
            redis.setex(cacheKey, 1800, JSON.toJSONString(product));
            return product;
        } else {
            // 获取锁失败，稍后重试
            Thread.sleep(50);
            return getProduct(id);  // 递归重试
        }
    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

2. **逻辑过期**：不设置 Redis 的 TTL，而是在 value 中附带一个过期时间字段：

```java
class CacheData {
    Product data;
    long expireTime;  // 逻辑过期时间戳
}

// 读时发现过期了，不阻塞，异步刷新
public Product getProduct(Long id) {
    CacheData cached = redis.get(cacheKey);
    if (cached == null || cached.expireTime < System.currentTimeMillis()) {
        // 异步刷新缓存，不阻塞当前请求
        asyncRefreshCache(id);
        // 返回旧数据（如果有）
        return cached != null ? cached.data : loadFromDb(id);
    }
    return cached.data;
}
```

### 5.3 缓存雪崩

**问题**：大量缓存在同一时间过期，或 Redis 整体宕机，请求全部涌向 DB。

**解决方案：**

1. **过期时间加随机值**：避免同时过期

```java
// 基础过期时间 + 随机偏移
int expireSeconds = 1800 + new Random().nextInt(300);  // 1800~2100 秒
redis.setex(key, expireSeconds, value);
```

2. **多级缓存**：本地缓存 + Redis + DB

```
请求 → Caffeine(本地) → Redis → DB
```

3. **限流降级**：用 Sentinel/Hystrix 限制 DB 的访问频率

---

## 六、方案对比与选型建议

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|----------|
| 先更 DB 再删缓存 | 短暂不一致（毫秒级） | 高 | 低 | 大多数业务场景 |
| 延迟双删 | 较小不一致窗口 | 中 | 中 | 对一致性有一定要求 |
| Canal + MQ 异步淘汰 | 最终一致（秒级） | 高 | 高 | 中大型项目，数据量大 |
| 强一致（同步双写） | 强一致 | 低 | 高 | 极少场景，不推荐 |

**选型建议：**

```
一致性要求不高（容忍几秒不一致）
  → 先更 DB 再删缓存 + TTL 兜底

一致性要求较高（容忍几百毫秒不一致）
  → 延迟双删

一致性要求很高（需要可追溯、可补偿）
  → Canal + MQ 异步淘汰 + 定时补偿

强一致要求（金融级）
  → 不要用缓存，直接读 DB
  → 或使用支持事务的缓存（如 EhCache 事务模式）
```

---

## 七、总结

缓存一致性的核心就三句话：

1. **写操作**：先更新 DB，再删除缓存。永远不要更新缓存，只删。
2. **兜底**：无论用什么方案，给缓存设过期时间是最基本的安全网。
3. **读操作**：用分布式锁解决缓存击穿，布隆过滤器解决缓存穿透，随机过期时间解决缓存雪崩。

没有银弹。缓存和 DB 之间天然存在不一致窗口，关键是根据业务场景选择**可接受的不一致程度**，然后用合适的方案把不一致的时间窗口控制在可接受范围内。
