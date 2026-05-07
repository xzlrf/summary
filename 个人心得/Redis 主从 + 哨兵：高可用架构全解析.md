# Redis 主从 + 哨兵：高可用架构全解析

> 一文搞懂 Redis 主从同步、哨兵故障转移机制

## 目录

- [一、为什么需要主从架构](#一为什么需要主从架构)
- [二、主从同步原理](#二主从同步原理)
  - [2.1 全量同步](#21-全量同步)
  - [2.2 增量同步](#22-增量同步)
  - [2.3 同步流程全景图](#23-同步流程全景图)
  - [2.4 断线重连](#24-断线重连)
  - [2.5 无盘复制（Diskless Replication）](#25-无盘复制diskless-replication)
- [三、主从架构的问题](#三主从架构的问题)
- [四、哨兵 Sentinel](#四哨兵-sentinel)
  - [4.1 哨兵的核心作用](#41-哨兵的核心作用)
  - [4.2 哨兵架构](#42-哨兵架构)
  - [4.3 主观下线（SDOWN）](#43-主观下线sdown)
  - [4.4 客观下线（ODOWN）](#44-客观下线odown)
  - [4.5 Leader 选举](#45-leader-选举)
  - [4.6 故障转移流程](#46-故障转移流程)
- [五、生产环境配置示例](#五生产环境配置示例)
- [六、常见问题与避坑](#六常见问题与避坑)
- [七、总结](#七总结)

---

## 一、为什么需要主从架构

单机 Redis 有两个致命问题：

1. **单点故障**：Redis 进程挂了就全完了
2. **性能瓶颈**：单机读写都在一个进程上，QPS 有上限
3. **容量瓶颈**：单个节点的内存容量有限

主从复制（Master-Slave）是最基础的解决方案：

```
        ┌──────────┐
        │  Master  │  ← 负责写
        │  (写)    │
        └────┬─────┘
             │ 异步复制
       ┌─────┴─────┐
       ▼           ▼
┌──────────┐ ┌──────────┐
│ Slave 1  │ │ Slave 2  │  ← 负责读
│  (读)    │ │  (读)    │
└──────────┘ └──────────┘
```

- **Master**：只负责写，数据变更同步给 Slave
- **Slave**：只读，分担读压力
- 一个 Master 可以有多个 Slave，Slave 也可以有自己的 Slave（链式复制）

---

## 二、主从同步原理

### 2.1 全量同步（Full Resynchronization）

全量同步发生在以下场景：
- Slave 首次连接 Master
- 断线时间过长，增量同步无法弥补
- 主从版本号不匹配

**全量同步的 6 个步骤：**

```
① Slave 发送 PSYNC 命令
       ↓
② Master 收到后，开始执行 BGSAVE（后台生成 RDB）
   同时，将新到的写命令缓存到 replication buffer
       ↓
③ BGSAVE 完成，Master 将 RDB 文件发送给 Slave
       ↓
④ Slave 收到 RDB，清空旧数据，加载 RDB 恢复数据
       ↓
⑤ Master 将 replication buffer 中缓存的写命令逐条发给 Slave
       ↓
⑥ Slave 执行这些命令，主从数据一致 → 进入正常同步阶段
```

**关键细节：**

- `BGSAVE` 是 fork 子进程进行的，不阻塞主线程（但 fork 瞬间会卡顿）
- `replication buffer` 是环形缓冲区，如果缓存的命令太多、Slave 太慢，可能被写满 → 只能触发新一轮全量同步
- RDB 传输期间 Master 仍在处理写请求，这些增量命令会在 RDB 传输完后补发给 Slave

**RDB 文件怎么传？**

Redis 4.0+ 支持两种传输方式：

| 方式 | 说明 | 适用场景 |
|------|------|----------|
| 有盘复制 | BGSAVE 生成 RDB 到磁盘，然后读取磁盘发送给 Slave | 默认方式，稳定 |
| 无盘复制 | BGSAVE 直接写入 socket，不写磁盘 | Slave 多且磁盘 IO 紧张时 |

### 2.2 增量同步（Partial Resynchronization）

全量同步代价很高（BGSAVE、RDB 传输、加载都很重）。Redis 2.8+ 引入了增量同步机制。

**核心数据结构：repl_backlog（复制积压缓冲区）**

```
┌─────────────────────────────────────────┐
│         repl_backlog（环形缓冲区）       │
│                                         │
│  offset=100     offset=200     offset=300│
│     │              │              │     │
│     ▼              ▼              ▼     │
│  [SET k1] ── [HSET k2] ─── [DEL k3]    │
│                                         │
│  大小由 repl-backlog-size 配置，默认 1MB │
└─────────────────────────────────────────┘
```

每个 Master 维护一个 `repl_backlog`，记录最近发送过的写命令。每个 Slave 记录自己的 `replication offset`（已经同步到哪个位置）。

**增量同步的判断逻辑：**

```
Slave 断线后重连，发送 PSYNC <runid> <offset>

Master 收到后：
  ├─ runid 不匹配 → 换主了，必须全量同步
  ├─ offset 超出 backlog 范围（数据已被覆盖）→ 全量同步
  └─ offset 在 backlog 范围内 → 增量同步
       把 offset 之后的命令发给 Slave，跳过全量同步
```

**为什么 repl_backlog 默认只有 1MB？**

这是一个 tradeoff。backlog 越大，能容忍的断线时间越长，但内存占用也越大。生产环境建议根据写操作频率调整：

```
backlog 大小 = 写 QPS × 单条命令平均大小 × 最大允许断线时间

# 例如：1000 QPS × 100 bytes × 60 秒 ≈ 6MB
repl-backlog-size 10mb
```

### 2.3 同步流程全景图

```
┌──────────────┐              ┌──────────────┐
│    Slave     │              │    Master    │
└──────┬───────┘              └──────┬───────┘
       │                             │
       │  ① PSYNC ? -1（首次连接）    │
       │────────────────────────────>│
       │                             │
       │  ② 启动 BGSAVE              │
       │  ③ 缓存后续写命令            │
       │                             │
       │  ④ 返回 FULLRESYNC + RDB    │
       │<────────────────────────────│
       │                             │
       │  ⑤ 清空旧数据，加载 RDB      │
       │                             │
       │  ⑥ 接收增量命令缓冲区         │
       │<────────────────────────────│
       │  ⑦ 执行增量命令              │
       │                             │
       │  ─── 进入正常同步阶段 ───     │
       │                             │
       │  每个写命令异步复制给 Slave   │
       │<────────────────────────────│
       │  ACK 回复 replication offset │
       │────────────────────────────>│
```

### 2.4 断线重连

```
Slave 断线 → 尝试重连
  │
  ├─ PSYNC <master_runid> <offset>
  │    │
  │    ├─ runid 匹配 & offset 在 backlog 内
  │    │   → 增量同步（CONTINUE）
  │    │
  │    └─ runid 不匹配 或 offset 过期
  │        → 全量同步（FULLRESYNC）
```

### 2.5 无盘复制（Diskless Replication）

当 Slave 数量很多时，每个 Slave 全量同步都需要 Master 生成一份 RDB 文件写入磁盘，磁盘 IO 压力巨大。

Redis 4.0+ 支持无盘复制：Master 直接通过 socket 将 RDB 发送给 Slave，不经过磁盘。

```
# 开启无盘复制
repl-diskless-sync yes

# 延迟几秒等待更多 Slave 连接，一次性多发
repl-diskless-sync-delay 5
```

---

## 三、主从架构的问题

主从复制解决了读扩展和高可用的问题，但有一个致命缺陷：

**Master 挂了，不会自动恢复。**

```
        ┌──────────┐
        │  Master  │  ← 宕机！
        └────┬─────┘
             │ 断连
       ┌─────┴─────┐
       ▼           ▼
┌──────────┐ ┌──────────┐
│ Slave 1  │ │ Slave 2  │  ← 都是只读，无法写入
└──────────┘ └──────────┘
```

这时候需要手动操作：
1. 选一个 Slave，执行 `SLAVEOF NO ONE` 把它提升为 Master
2. 让其他 Slave 指向新 Master
3. 应用层修改 Master 连接地址

**这太痛苦了，而且慢。** 于是哨兵（Sentinel）登场。

---

## 四、哨兵 Sentinel

### 4.1 哨兵的核心作用

Sentinel 是 Redis 自带的**高可用解决方案**，它不存储数据，专门负责监控和故障转移：

1. **监控**：定时检查 Master 和 Slave 是否正常运行
2. **通知**：当某个 Redis 实例出问题时，通过 API 通知管理员或其他应用
3. **故障转移**：Master 宕机后，自动选一个 Slave 升级为 Master，其他 Slave 指向新 Master
4. **配置中心**：客户端连接 Sentinel 获取当前 Master 地址，不用硬编码

### 4.2 哨兵架构

```
                    ┌──────────┐
                    │ Sentinel1│ ◄── 领导者
                    └────┬─────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Sentinel2│ │ Sentinel3│ │ Sentinel4│
        └──────────┘ └──────────┘ └──────────┘
              │          │          │
              ▼          ▼          ▼
        ┌─────────────────────────────┐
        │   Redis 主从集群            │
        │   Master + Slave1 + Slave2  │
        └─────────────────────────────┘
```

**关键原则**：Sentinel 至少部署 **3 个节点**（奇数），通过 Raft 算法选举 Leader 来做故障转移决策。

### 4.3 主观下线（SDOWN）

**定义**：单个 Sentinel 实例对某个 Redis 节点判定为"下线"。

**检测机制**：

每个 Sentinel 每隔 **1 秒**向所有已知的 Redis 实例发送 `PING` 命令：

```
Sentinel  ─── PING ───▶  Redis 实例
           ◀── PONG ───  （正常响应）
```

**触发 SDOWN 的条件：**

如果超过 `down-after-milliseconds` 毫秒（默认 30000ms = 30 秒）没有收到有效回复（PONG、-LOADING、-MASTERDOWN），Sentinel 就会主观认为该实例下线。

```
# 配置文件
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
```

**重要理解**：SDOWN 只是**单个 Sentinel 的主观判断**，不代表整个 Sentinel 集群都这么认为。就像你一个人觉得某人没来上班，可能是他自己请假了，不代表他真出事了。

### 4.4 客观下线（ODOWN）

**定义**：多个 Sentinel 实例一致同意某个 Master 下线。

**触发流程：**

```
Sentinel A 发现 Master 没响应
  → 标记 Master 为 SDOWN（主观下线）
  → 向其他 Sentinel 发送 is-master-down-by-addr 询问
  → 收到足够多的 "同意下线" 回复
  → 数量 ≥ quorum（配置中的法定人数）
  → 标记 Master 为 ODOWN（客观下线）
```

**quorum 是什么？**

在 `sentinel monitor` 配置中指定：

```
sentinel monitor mymaster 127.0.0.1 6379 2
                                            ↑
                                       quorum = 2
```

意思是：至少需要 2 个 Sentinel 同意，才能判定 Master 客观下线。

**SDOWN 和 ODOWN 的区别总结：**

| 特性 | SDOWN（主观下线） | ODOWN（客观下线） |
|------|-------------------|-------------------|
| 判定者 | 单个 Sentinel | 多个 Sentinel 协商 |
| 适用对象 | Master 和 Slave | 仅 Master |
| 触发条件 | 超时未回复 | 同意下线的数量 ≥ quorum |
| 结果 | 记录状态，继续观察 | 触发 Leader 选举和故障转移 |

### 4.5 Leader 选举

当 Master 被标记为 ODOWN 后，Sentinel 集群需要选出一个 **Leader** 来执行故障转移。

**选举算法**：Raft Leader Election

```
Sentinel A 发现自己认为 Master 下线了
  → 向其他 Sentinel 发送 is-master-down-by-addr 请求
  → 请求中声明 "我想当 Leader，纪元(epoch)=X"
  → 每个 Sentinel 在同一纪元内只投一票
  → 获得大多数票（> N/2）的 Sentinel 成为 Leader
```

**选举失败怎么办？**

如果没有 Sentinel 获得大多数票（比如网络分区），本轮选举失败。等待一段时间后重新选举。最终一定能选出 Leader。

### 4.6 故障转移流程

选出了 Leader 之后，故障转移正式开始：

```
步骤 1：从 Slave 中选一个新的 Master
  │
  ├─ 过滤掉线、响应慢的 Slave
  ├─ 优先级最高的 Slave 优先（slave-priority）
  ├─ 优先级相同，选复制偏移量最大的（数据最新的）
  └─ 还相同，选 runid 最小的
  │
步骤 2：让新 Master 执行 SLAVEOF NO ONE
  │  → 它从 Slave 变成 Master，可以接受写了
  │
步骤 3：让其他 Slave 指向新 Master
  │  → 发送 SLAVEOF <new-master-ip> <new-master-port>
  │
步骤 4：更新 Sentinel 自己的配置
  │  → 记录新 Master 的地址
  │
步骤 5：等待旧 Master 恢复
  └─ 旧 Master 恢复后，自动变成新 Master 的 Slave
```

**整个过程耗时大约**：SDOWN 超时（默认 30s）+ 协商 ODOWN（几秒）+ Leader 选举（几秒）+ 切换主从（几秒）= **大约 30~40 秒**。

生产环境建议将 `down-after-milliseconds` 调小（比如 5000ms = 5 秒）：

```
sentinel down-after-milliseconds mymaster 5000
```

---

## 五、生产环境配置示例

### 5.1 主从配置

**Master (6379)：**
```
# redis.conf - Master
port 6379
bind 0.0.0.0
requirepass yourpassword
masterauth yourpassword
```

**Slave (6380)：**
```
# redis.conf - Slave1
port 6380
bind 0.0.0.0
requirepass yourpassword
masterauth yourpassword
replicaof 192.168.1.100 6379
```

**Slave (6381)：**
```
# redis.conf - Slave2
port 6381
bind 0.0.0.0
requirepass yourpassword
masterauth yourpassword
replicaof 192.168.1.100 6379
```

### 5.2 哨兵配置

三个 Sentinel 节点配置相同（部署在不同机器上）：

```
# sentinel.conf
port 26379

# 监控 Master，quorum = 2
sentinel monitor mymaster 192.168.1.100 6379 2

# 3 秒无响应判定主观下线
sentinel down-after-milliseconds mymaster 3000

# 故障转移超时
sentinel failover-timeout mymaster 10000

# 同时同步的 Slave 数量（限制新 Master 的并发压力）
sentinel parallel-syncs mymaster 1

# 认证
sentinel auth-pass mymaster yourpassword
```

**启动哨兵：**
```bash
redis-sentinel /path/to/sentinel.conf
# 或
redis-server /path/to/sentinel.conf --sentinel
```

### 5.3 Java 客户端连接哨兵

```java
Set<String> sentinels = new HashSet<>();
sentinels.add("192.168.1.101:26379");
sentinels.add("192.168.1.102:26379");
sentinels.add("192.168.1.103:26379");

JedisSentinelPool pool = new JedisSentinelPool(
    "mymaster",      // 哨兵监控的 Master 名称
    sentinels,
    "yourpassword"
);

Jedis jedis = pool.getResource();
// 自动连接到当前 Master
jedis.set("key", "value");
```

---

## 六、常见问题与避坑

### 坑 1：脑裂导致数据丢失

**场景**：Master 网络分区，Sentinel 选出新 Master。旧 Master 网络恢复后，客户端可能还在往旧 Master 写数据。等旧 Master 变成 Slave 时，这些数据全部丢失。

**缓解方案**：

```
# 限制 Master 的最小连接 Slave 数量
min-replicas-to-write 1

# Slave 最大延迟（秒），超过就不认这个 Slave
min-replicas-max-lag 10
```

### 坑 2：Slave 全挂了，Sentinel 不会切换

Sentinel 只在 Master 下线时才做故障转移。如果所有 Slave 都挂了但 Master 还活着，Sentinel 不会做任何事情。

### 坑 3：down-after-milliseconds 设置太短

设太短（比如 1000ms），网络抖动就可能触发不必要的故障转移。建议至少 3000ms~5000ms。

### 坑 4：哨兵节点数太少

如果只部署 1 个 Sentinel，它本身就是单点故障。如果部署 2 个，quorum 最多设为 2，任何一个挂了就无法做故障转移。**最少 3 个，推荐 5 个**（容忍 2 个节点宕机）。

### 坑 5：parallel-syncs 设置不当

```
sentinel parallel-syncs mymaster 1
```

这表示故障转移时，同时只有 1 个 Slave 向新 Master 同步。设为 1 比较安全，但切换期间所有 Slave 都会暂时不可用。如果设为更大的值，切换更快，但新 Master 压力更大。

---

## 七、总结

### 主从同步核心要点

| 知识点 | 要点 |
|--------|------|
| 全量同步 | BGSAVE 生成 RDB → 传输 → Slave 加载 → 补增量命令 |
| 增量同步 | repl_backlog 环形缓冲区，基于 offset 判断是否需要全量 |
| 同步时机 | 连接时全量，之后增量 |
| 无盘复制 | Socket 直接传 RDB，不写磁盘，适合多 Slave 场景 |

### 哨兵核心要点

| 知识点 | 要点 |
|--------|------|
| SDOWN | 单个 Sentinel 主观判断，超时未 PONG |
| ODOWN | 多个 Sentinel 协商一致，数量 ≥ quorum |
| Leader 选举 | Raft 算法，大多数票胜出 |
| 故障转移 | 选新 Master → 切换主从 → 更新配置 → 旧 Master 变 Slave |
| 推荐部署 | 3~5 个 Sentinel 节点，奇数个 |

### 一句话总结

主从复制解决了**数据冗余和读扩展**，哨兵解决了**自动故障转移**，两者配合才能构建一个生产可用的 Redis 高可用架构。但要注意：这套方案不解决**写扩展**（写仍然集中在一个 Master），如果写压力也大，需要上 Redis Cluster（分片集群）。
