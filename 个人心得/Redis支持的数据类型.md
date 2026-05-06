### Redis数据类型

- 通用/基础类型：

  - String（字符串，也常作为计数器、位图等使用）
  - Hash（哈希/对象）
  - List（列表）
  - Set（集合）
  - Sorted Set（有序集合，也常叫 ZSet）
  - Stream（流，常用于日志/事件/消息）
  - JSON（结构化 JSON 文档）

- 扩展/模块类型：

  - Vector Set（向量集合，向量相似检索）
  - Geospatial（地理空间索引）
  - Bitmaps / Bitfields（位图、位字段）
  - Probabilistic data types（概率型数据结构：HyperLogLog、Bloom、Cuckoo filter、t-digest、Top-K、Count-min sketch）
  - Time Series（时间序列）

  #### 类型如何选择？

  - 存“文档/对象”（如用户信息、商品信息）
    - 有嵌套结构、需要 Redis Search 复杂查询 → JSON
    - 字段不多、要更省内存、字段级操作/过期 → Hash
    - 对象较小且整体读写为主，只需整体读写 → String（JSON 序列化后存储）
  - 存“集合”并做去重/交并差
    - 需要排序/按分值范围取值 → Sorted Set
    - 只需要去重、交集/并集等 → Set
    - 键是整数区间、需要位运算 → String（用位图）
  - 存“队列/流式数据”
    - 需要消费组、至少一次语义、按时间顺序追加读取 → Stream
    - 简单 FIFO/LIFO 队列、栈 → List
    - 需要按优先级/顺序+按分值取→ Sorted Set（例如优先级队列）

## 最常用类型：典型命令与场景速查

### 1）String（字符串）

常用命令：

- 读写：`SET key value`、`GET key`、`MGET key1 key2`
- 计数：`INCR key`（原子自增）、`INCRBY key step`、`DECR key`
- 生存时间（通用）：`EXPIRE key seconds`、`TTL key`、`PERSIST key`
- 删除/存在性：`DEL key [key …]`、`EXISTS key`
- 追加/长度：`APPEND key value`、`STRLEN key`
- 过期写法（写时设 TTL）：`SET key value EX seconds` 或 `SETEX key seconds value`

典型场景：

- 缓存：热点数据、页面片段、接口结果缓存（`SET` + `EX`）
- 计数器：点赞数、播放量、限流计数（`INCR`/`INCRBY`）
- Session / Token 存储：`SET session:xxx <json> EX 1800`
- 分布式锁（简单版）：`SET lock:xxx 1 NX EX 10`（实际常配合 Lua 或 Redisson 使用）redis.io
- Bitmaps 应用：日活、在线状态、签到等（把字符串当作位数组来用，命令如 `SETBIT/GETBIT/BITCOUNT/BITOP`）

### 2）Hash（哈希/对象）

常用命令：

- 单字段：`HSET key field value`、`HGET key field`
- 多字段：`HMGET key field1 field2 …`
- 全量/增删：
  - `HGETALL key`（获取全部字段和值）
  - `HDEL key field [field …]`
  - `HEXISTS key field`
  - `HINCRBY key field step`
- 字段级过期（较新的版本支持）：`HEXPIRE key seconds FIELDS field1 field2 …`

典型场景：

- 存储对象：用户资料、商品属性（不需要嵌套、字段数适中的情况）
- 计数器组：同一对象多个计数器（点赞数、评论数、分享数等字段，用 `HINCRBY`）
- 需要字段级 TTL 的场景：比如每个属性独立的过期时间（利用字段级过期）
- 与 Redis Search 联合做索引/查询（字段级索引与过滤）

### 3）List（列表）

常用命令：

- 两端压入/弹出：
  - 左：`LPUSH key value [value …]`、`LPOP key`
  - 右：`RPUSH key value [value …]`、`RPOP key`
- 阻塞弹出（常用于队列）：`BLPOP key [key …] timeout`、`BRPOP key [key …] timeout`
- 范围/长度：
  - `LRANGE key start stop`
  - `LLEN key`
  - `LINDEX key index`
  - `LTRIM key start stop`（保留区间，常做固定长度队列）

典型场景：

- 简单消息队列/任务队列（FIFO 或栈）
- 最新列表（如“最新文章/Twitter 时间线”），用 `LPUSH + LTRIM`
- 简单的排队/先进先出（生产者–消费者）

注意：Redis List 更适合“头尾操作”的队列；频繁在中间插入/删除性能较差，官方也推荐 Stream 做更健壮的消息模型。

### 4）Set（集合）

常用命令：

- 增删查：
  - `SADD key member [member …]`
  - `SREM key member [member …]`
  - `SISMEMBER key member`
  - `SCARD key`、`SMEMBERS key`
  - `SRANDMEMBER key [count]`（随机取样）
  - `SPOP key [count]`
- 集合运算：
  - `SINTER key1 key2`、`SUNION key1 key2`、`SDIFF key1 key2`

典型场景：

- 去重（如已读、已点赞、已领取 ID 集合）
- 标签系统（用户–标签、文章–标签的多对多映射）
- 共同好友/共同关注（`SINTER`）
- 随机抽奖（`SRANDMEMBER`/`SPOP`）

### 5）Sorted Set（ZSet / 有序集合）

常用命令：

- 添加/更新与分值：`ZADD key score member [score member …]`
- 范围查询（按分值/按排名）：
  - `ZRANGE key start stop [BYSCORE] [WITHSCORES] [REV] [LIMIT offset count]`
  - `ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]`
- 删除与数量：
  - `ZREM key member [member …]`
  - `ZCARD key`、`ZSCORE key member`、`ZRANK key member`（按升序排名）、`ZREVRANK key member`
- 增减分值：`ZINCRBY key increment member`
- 交集/并集：`ZINTERSTORE`、`ZUNIONSTORE`（可用于多维度排行聚合）

典型场景：

- 排行榜：全局排行榜、天/周/月排行（按分值倒序取前 N）
- 优先级队列/延迟队列（以时间戳为 score）
- 带权重的标签/计数（例如热词、热度分、评分）
- 范围型负载均衡/评分路由（按分值区间选择实例）