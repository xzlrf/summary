## MVCC 原理

### 是什么

**MVCC，全称 Multi-Version Concurrency Control，即多版本并发控制。**

核心思想：**读不加锁，读写不冲突。** 读操作读取历史版本，写操作修改当前版本，两者互不干扰。

MySQL 中同一行数据会存在多个版本，MVCC 让不同事务根据隔离级别看到合适的版本。

### MVCC 解决了什么问题

| 并发问题 | 描述 | MVCC 是否解决 |
|----------|------|--------------|
| 脏读 | 读到别人未提交的数据 | **解决**（只读已提交版本） |
| 不可重复读 | 同一事务两次读到不同数据 | **RR 级别下解决**（同一个事务用同一个 Read View） |
| 写-写冲突 | 两个事务同时修改一行 | **不解决**（靠行锁解决） |

一句话：**MVCC 解决了读-写冲突，让读操作不需要加锁也能看到一致的数据。**

### 底层实现：三个核心组件

#### 1. 隐式字段

每行数据除了我们定义的列，InnoDB 还藏了三个字段：

| 字段 | 含义 |
|------|------|
| `DB_ROW_ID` | 隐藏主键，没定义主键时自动生成 |
| `DB_TRX_ID` | 最后一次修改该行数据的**事务 ID** |
| `DB_ROLL_PTR` | **回滚指针**，指向 undo log 中的旧版本记录 |

#### 2. undo log 版本链

每次 UPDATE/DELETE 时，InnoDB 会先把旧值写入 undo log，然后用 `DB_ROLL_PTR` 串起来。

```
最新数据 (DB_TRX_ID=103, DB_ROLL_PTR -> undo2)
    ↓
undo2  (DB_TRX_ID=102, DB_ROLL_PTR -> undo1)  -- 事务102改的
    ↓
undo1  (DB_TRX_ID=101, DB_ROLL_PTR -> undo0)  -- 事务101改的
    ↓
undo0  (DB_TRX_ID=100, DB_ROLL_PTR -> null)   -- 初始值
```

**不同事务对同一行的修改，通过 undo log 形成了一条从新到旧的版本链。**

#### 3. Read View（一致性视图）

Read View 是事务在某个时刻拍的一张"快照"，用来判断版本链中哪个版本对当前事务可见。

**Read View 包含四个关键信息**：

| 字段 | 含义 |
|------|------|
| `m_ids` | 创建 Read View 时**活跃（未提交）的事务 ID 列表** |
| `min_trx_id` | `m_ids` 中的**最小**事务 ID |
| `max_trx_id` | 创建 Read View 时**下一个要分配的事务 ID**（全局最大+1） |
| `creator_trx_id` | 创建这个 Read View 的**事务自身 ID** |

**可见性判断规则**（拿一行数据的 `DB_TRX_ID` 和 Read View 对比）：

```
1. DB_TRX_ID < min_trx_id
   -> 事务在 Read View 创建之前就已经提交了 -> 可见

2. DB_TRX_ID >= max_trx_id
   -> 事务在 Read View 创建之后才启动 -> 不可见，继续往旧版本找

3. min_trx_id <= DB_TRX_ID < max_trx_id
   -> 事务在 Read View 创建时是活跃的
   -> 如果 DB_TRX_ID 在 m_ids 中（未提交）-> 不可见
   -> 如果 DB_TRX_ID 不在 m_ids 中（已提交）-> 可见

4. DB_TRX_ID == creator_trx_id
   -> 自己修改的 -> 可见
```

**简单记法**：
- 小于 min：已提交，**可见**
- 大于等于 max：还没开始，**不可见**
- 在中间范围：看 m_ids 列表，在列表里就是没提交，**不可见**；不在列表里就是已提交，**可见**

### RC vs RR 级别下 Read View 的生成时机

这是 MVCC 最核心的区别：

| 隔离级别 | Read View 生成时机 | 效果 |
|----------|-------------------|------|
| **RC（读已提交）** | **每次 SELECT 都生成新的 Read View** | 能读到其他事务新提交的数据（不可重复读） |
| **RR（可重复读）** | **整个事务只在第一次 SELECT 时生成一次 Read View** | 整个事务看到的数据始终一致（可重复读） |

**举例说明区别**：

```
初始：balance = 100

事务 A 开启（trx_id = 100）
事务 B 开启（trx_id = 101）

事务 B: UPDATE users SET balance = 200; COMMIT;

RC 级别：
  事务 A 第一次 SELECT -> balance = 100（B 还没提交）
  事务 A 第二次 SELECT -> 重新生成 Read View -> balance = 200（B 已提交）
  -> 不可重复读！

RR 级别：
  事务 A 第一次 SELECT -> 生成 Read View -> balance = 100
  事务 A 第二次 SELECT -> 复用同一个 Read View -> balance = 100
  -> 可重复读！
```

### 快照读 vs 当前读

| 类型 | 说明 | SQL 示例 | 是否加锁 | 读什么版本 |
|------|------|----------|----------|-----------|
| **快照读** | 读取快照数据，不加锁 | 普通 `SELECT` | 否 | 历史版本（通过 Read View + undo log） |
| **当前读** | 读取最新数据，加锁 | `SELECT ... FOR UPDATE`、`SELECT ... LOCK IN SHARE MODE`、`UPDATE`、`DELETE`、`INSERT` | 是 | 最新版本（加锁保证不被修改） |

```
快照读 = 乐观锁实现（读不加锁，靠 MVCC 保证一致性）
当前读 = 悲观锁实现（读加锁，阻塞其他写操作）
```

**注意**：快照读的前提是隔离级别不是串行化。在串行化级别下，事务之间完全串行执行，快照读会退化为当前读。

### RR 级别如何避免幻读

```
RR 级别下，普通 SELECT（快照读）靠 MVCC 避免幻读：
  因为整个事务共用同一个 Read View，
  其他事务插入的新记录 DB_TRX_ID >= max_trx_id，对当前事务不可见。
  所以两次 SELECT 看到的记录数量一致 -> 没有幻读。

但当前读（SELECT ... FOR UPDATE）MVCC 不管用：
  因为当前读读的是最新版本，其他事务插入的记录对它是可见的。
  所以 RR 级别下，当前读的幻读靠**间隙锁（Gap Lock）+ 临键锁（Next-Key Lock）**来避免。
```

总结：

| 读类型 | 避免幻读的方式 |
|--------|---------------|
| 快照读（普通 SELECT） | MVCC（Read View + undo log 版本链） |
| 当前读（FOR UPDATE / UPDATE / INSERT） | 间隙锁 + 临键锁 |
