## MySQL 索引与执行计划

[TOC]

### 一、聚簇索引（主键索引）

#### 聚簇索引是什么

聚簇索引（clustered index）不是一种"特殊的索引类型"，而是**"数据本身按某个索引键值的顺序存储"的一种存储方式**。

在 InnoDB 里，这个"按哪一列排好序、把整行数据放在叶子节点"的索引，就叫聚簇索引。如果表有主键，主键索引通常就是聚簇索引。可以简单理解为：

- 聚簇索引 = **"按某个键排好序的 B+ 树，叶子节点直接放的是整行数据"**
- 其他（二级/非聚簇）索引 = **"另一棵 B+ 树，叶子放的是索引列 + 主键"**

**如果没定义主键会怎样？**

1. 如果你没定义主键，但有一个"所有列都 NOT NULL 的 UNIQUE 索引"：InnoDB 会用第一个这样的唯一索引作为聚簇索引
2. 如果既没有主键，也没有合适的唯一索引：InnoDB 会内部生成一个隐藏列（row_id），并在其上建立一个隐藏的聚簇索引 `GEN_CLUST_INDEX`

**"聚簇索引"和"主键索引"是什么关系？**

严格讲：
- **主键（PRIMARY KEY）是一个约束**：唯一 + 非空
- **聚簇索引是一种索引/存储方式**：决定数据怎么存储

主键索引通常是聚簇索引，但聚簇索引不一定是主键索引（比如用了 NOT NULL UNIQUE 索引作为聚簇索引的情况）。

#### 叶子节点存的是整行数据

聚簇索引的叶子节点存储的是**完整的行数据**（所有列的值），而不是只存索引键值。这就是为什么通过主键查询可以直接拿到完整数据，不需要额外查询。

#### 非聚簇索引（二级索引）叶子节点存的是主键值

非聚簇索引（Secondary Index）的叶子节点存的是：**索引列值 + 主键值**。

例如在 `username` 列上建了二级索引，叶子节点存的是 `(username, id)`，其中 `id` 是主键。

#### InnoDB 必须有聚簇索引，MyISAM 没有

| 引擎 | 聚簇索引 | 数据与索引的关系 |
|------|----------|-----------------|
| InnoDB | 必须有 | 数据文件本身就是聚簇索引（索引即数据） |
| MyISAM | 没有 | 数据和索引是分开存储的（.MYD 存数据，.MYI 存索引） |

MyISAM 的所有索引（包括主键）都是非聚簇索引，叶子节点存的是数据文件的物理地址（指针），而不是主键值。

#### 聚簇索引和非聚簇索引对比

| 维度 | 聚簇索引（主键索引） | 非聚簇索引（二级索引） |
|------|---------------------|----------------------|
| 叶子节点存什么 | 整行数据 | 索引列值 + 主键值 |
| 一棵表有几个 | 只能有 1 个 | 可以有多个 |
| 查询是否需要回表 | 不需要（叶子就是数据） | 可能需要（回表查聚簇索引） |
| 数据排列方式 | 按索引键值顺序物理存储 | 按索引键值排序，但数据不跟着排 |
| 覆盖查询 | 天然覆盖（所有列都在叶子） | 只有查询列都在索引中时才覆盖 |

### 二、回表查询

#### 回表是什么、为什么会慢

回表就是**拿着二级索引叶子节点中的主键值，再去聚簇索引中把完整行数据取出来**。

```
SELECT * FROM users WHERE username = 'zhangsan';

执行过程：
1. 在 username 二级索引树中找到 ('zhangsan', id=5)
2. 拿 id=5 去聚簇索引树中查找完整行数据  <- 这就是回表
3. 返回所有列
```

**为什么慢？**
- 相当于多查了一次 B+ 树
- 回表本质是随机 I/O（从二级索引跳到聚簇索引的不同位置）
- 如果回表次数多（比如查 1 万条记录），会产生大量随机 I/O，性能骤降

#### 什么情况下会发生回表

查询走了二级索引，并且 `SELECT` 的字段不在索引覆盖范围内，就会导致回表。

#### 索引下推（ICP）减少回表

MySQL 5.6+ 引入了**索引下推（Index Condition Pushdown）**：把 `WHERE` 里能用"索引列"直接判断的那部分条件，下推到存储引擎层，在遍历索引时就过滤。只有满足这部分条件的记录，才去"回表"读整行，从而减少回表次数和 I/O。

```sql
-- 假设联合索引 (name, age)
SELECT * FROM users WHERE name LIKE '张%' AND age = 20;

没有 ICP：
  1. 二级索引找到所有 name LIKE '张%' 的记录（可能很多）
  2. 每条都回表拿到完整行
  3. 再过滤 age = 20

有 ICP：
  1. 二级索引找到 name LIKE '张%' 的记录
  2. 直接在索引层判断 age 是否为 20
  3. 只有 age = 20 的才回表
```

### 三、联合索引结构

联合索引（也叫复合索引）本质上是**"一棵 B+ 树，但它的排序键是多列组合成的元组 (a, b, c, …)"**。

- 叶子节点：按 `(col1, col2, …, colN)` 排好序的键值（在 InnoDB 里还带着主键值）
- 非叶子节点：同样按多列组合排序，用来做"范围/等值查找"的导航

正因为这棵树是"先按第 1 列排序，第 1 列相同时再按第 2 列排序……"，才有了**最左前缀原则**：只有从最左列开始连续的条件，才能直接用上这棵树的有序性来加速查找。

```sql
-- 联合索引 (a, b, c)
WHERE a = 1              -> 走索引
WHERE a = 1 AND b = 2    -> 走索引
WHERE a = 1 AND b = 2 AND c = 3  -> 走索引
WHERE a = 1 AND c = 3    -> 只用到 a（b 断了）
WHERE b = 2 AND c = 3    -> 不走索引（缺少最左列 a）
WHERE b = 2              -> 不走索引
```

### 四、为什么建议用自增主键

1. **减少页分裂**：InnoDB 的表就是按主键排好序存的（聚簇索引），自增能让插入几乎都落在"最后一个页"，减少页分裂，插入更快更稳定
2. **二级索引更小**：每个二级索引都会在叶子节点存一份主键。主键越短（整型比 UUID 小），所有索引越省空间，缓存越好，回表也越快
3. **比较速度快**：纯数字的整型在索引中的比较速度远比字符串快

```
自增 BIGINT UNSIGNED：8 字节
INT：4 字节
UUID 字符串（char(36)）：36 字节
UUID BINARY(16)：16 字节，但仍是字符串比较

主键越短 -> 二级索引越小 -> 内存能缓存的记录越多 -> 回表越快
```

### 五、EXPLAIN 核心字段（重点）

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid' ORDER BY create_time DESC;
```

执行计划返回的关键字段：

#### 5.1 type：访问类型（从差到好）

| type | 含义 | 示例 | 是否需要优化 |
|------|------|------|-------------|
| ALL | 全表扫描 | 没有索引，扫描每一行 | **必须优化** |
| index | 全索引扫描 | 扫描整个索引树，不扫描数据行 | 通常需要优化 |
| range | 范围扫描 | `WHERE id > 10`、`BETWEEN`、`IN`、`LIKE 'xxx%'` | 可接受 |
| ref | 非唯一索引等值匹配 | `WHERE user_id = 100`（user_id 是普通索引） | 良好 |
| eq_ref | 唯一索引等值匹配（JOIN 场景） | `JOIN` 中用主键关联 | 优秀 |
| const | 常量查询（主键/唯一索引等值） | `WHERE id = 1` | 最优 |
| system | const 的特例（表只有一行数据） | 系统表 | 极少见 |

**判断标准**：
- `ALL` 和 `index`：**必须优化**
- `range`：一般可接受，但如果扫描行数太多也要优化
- `ref` 及以上：很好

#### 5.2 key：实际用到的索引

- 显示 MySQL 优化器**实际选择的索引名**
- 如果为 `NULL`，说明没走任何索引（全表扫描）
- 可能和 `possible_keys` 不同（possible_keys 是候选索引，key 是最终选中的）

#### 5.3 key_len：索引使用长度

```
表示索引中实际使用的字节数。可以判断联合索引用到了几列。

示例：联合索引 (user_id, status, create_time)
  user_id INT NOT NULL     -> 4 字节
  status VARCHAR(10)       -> 10 * 3(utf8) + 2(变长) = 32 字节
  create_time DATETIME     -> 5 字节

EXPLAIN 结果 key_len = 4   -> 只用到了 user_id
EXPLAIN 结果 key_len = 36  -> 用到了 user_id + status
EXPLAIN 结果 key_len = 41  -> 三列都用到了
```

**判断技巧**：
- key_len 越长，说明联合索引用到的列越多
- 如果 key_len 短于预期，说明有部分索引列没用到

#### 5.4 rows：预估扫描行数

- 优化器估算的需要扫描的行数（不是精确值）
- 数值越小越好
- 如果 rows 接近总行数，说明索引效果差或没走索引

#### 5.5 Extra：额外信息（非常关键）

| Extra 值 | 含义 | 是否需要优化 |
|----------|------|-------------|
| **Using index** | 覆盖索引：所需数据全在索引中，**不需要回表** | 很好，不需要优化 |
| **Using where** | 存储引擎返回数据后，Server 层再用 WHERE 过滤 | 正常，但如果 rows 很大说明索引不够好 |
| **Using filesort** | 无法利用索引排序，需要额外的排序操作（内存或磁盘） | **需要优化** |
| **Using temporary** | 使用了临时表（常见于 GROUP BY / DISTINCT / UNION） | **需要优化** |

**四种 Extra 的组合判断**：

```
Using index                          -> 最优（覆盖索引，无回表）
Using index + Using where            -> 良好（覆盖索引 + 额外过滤）
Using where                          -> 一般（有回表，需要 WHERE 过滤）
Using where + Using filesort         -> 需要优化（无法用索引排序）
Using where + Using temporary        -> 需要优化（临时表开销大）
Using filesort + Using temporary     -> 需要优化（两个都占了）
```

#### 5.6 看执行计划判断是否需要优化的完整流程

```
1. 看 type
   ALL / index -> 没走索引或全索引扫描，必须优化
   range 及以下 -> 继续看

2. 看 key
   NULL -> 没走索引，必须优化
   有值 -> 继续看

3. 看 key_len
   短于预期 -> 联合索引没完全用到，考虑调整索引列顺序

4. 看 rows
   行数太多 -> 索引区分度不够，考虑加列或换索引

5. 看 Extra
   Using filesort -> 无法用索引排序，考虑建包含 ORDER BY 列的索引
   Using temporary -> 用了临时表，考虑优化 GROUP BY / DISTINCT
   Using index -> 覆盖索引，性能优秀
```

### 六、慢查询 & 简单优化

#### 6.1 开启慢查询日志

```sql
-- 查看当前慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';

-- 设置慢查询阈值（单位：秒，设置为 1 表示超过 1 秒的 SQL 记录）
SET GLOBAL long_query_time = 1;

-- 指定日志文件路径（可选）
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 永久配置（my.cnf / my.ini）
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = ON    -- 没走索引的 SQL 也记录
```

#### 6.2 分析慢查询

```bash
# 方式一：直接看慢查询日志
tail -f /var/log/mysql/slow.log

# 方式二：用 mysqldumpslow 工具统计分析
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
# -s t：按耗时排序
# -t 10：显示前 10 条

# 方式三：用 EXPLAIN 分析慢 SQL
EXPLAIN SELECT * FROM orders WHERE ...;
```

#### 6.3 典型优化手段

**优化一：加索引**

```sql
-- 慢 SQL
SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';
-- EXPLAIN 显示 type=ALL，rows=100000

-- 优化：加联合索引
CREATE INDEX idx_user_status ON orders(user_id, status);
-- EXPLAIN 显示 type=ref，rows=50
```

**优化二：避免大分页**

```sql
-- 慢：LIMIT 偏移量很大时，MySQL 要扫描前面所有行
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;

-- 优化 1：延迟关联（先查主键，再 JOIN 回表）
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 20) t
ON o.id = t.id;

-- 优化 2：游标分页（记住上一页最后一条的 id）
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 20;
```

**优化三：减少 SELECT ***

```sql
-- 慢：SELECT * 回表拿所有列，传输数据量大
SELECT * FROM users WHERE username = 'zhangsan';

-- 优化：只查需要的列，可能触发覆盖索引（Using index）
SELECT id, username, email FROM users WHERE username = 'zhangsan';
-- 如果有索引 (username, id, email)，则 Using index，不需要回表
```

### 七、什么是 Filesort

在 MySQL 中，`filesort` 只是一个**内部排序算法的名字**（名字起得很烂，极具误导性）。只要 MySQL 发现结果集需要排序，但**没有合适的索引**能直接提供这个顺序时，就会触发 `filesort` 操作。

总结：**需要排序，但索引帮不上忙，就会 filesort。**

- 如果结果集大小没超过 `sort_buffer_size`，在**内存**中排序
- 超过 `sort_buffer_size`，在**磁盘**排序

```
索引排序 >>> 内存 filesort >>> 磁盘 filesort
```

只要出现以下关键字，MySQL 就需要返回一个"有序的结果集"：

1. **`ORDER BY`**：最常见，显式要求排序
2. **`GROUP BY`**：分组计算前，必须先按分组字段排序（把相同的值聚在一起）
3. **`DISTINCT`**：去重，底层通常也是先排序再去除相邻重复行
4. **某些聚合函数或 JOIN 条件**：为了优化执行，内部可能需要先排序

**如何优化 filesort？**

1. 建立合适的联合索引，让索引顺序与 `ORDER BY` 顺序一致
2. 减小 SELECT 的数据量，只 SELECT 必要的字段
3. 加大排序缓冲区 `sort_buffer_size`
4. 利用 `LIMIT` 减少排序负担
