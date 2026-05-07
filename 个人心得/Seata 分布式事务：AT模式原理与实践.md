# Seata 分布式事务：AT 模式原理与实践

## 一、分布式事务问题

### 1.1 为什么需要分布式事务

```
单体应用：
  本地事务（@Transactional）-> 一个数据库 -> ACID 保证

微服务架构：
  服务 A（订单库）-> 创建订单
  服务 B（库存库）-> 扣减库存
  服务 C（账户库）-> 扣减余额

问题：三个操作分布在三个服务的三个数据库上，本地事务管不了。
      如果订单创建成功但库存扣减失败，数据就不一致了。
```

### 1.2 CAP 定理与 BASE 理论

**CAP 定理**：分布式系统最多同时满足三个特性中的两个。

| 特性 | 含义 | 说明 |
|------|------|------|
| C（Consistency） | 一致性 | 所有节点同一时刻的数据一致 |
| A（Availability） | 可用性 | 每个请求都能得到响应（不保证最新） |
| P（Partition Tolerance） | 分区容错性 | 网络分区故障时系统仍能运行 |

```
分布式系统中 P 是必然的（网络不可能 100% 可靠）
所以只能在 CP 和 AP 之间选择：

CP（强一致性）：ZooKeeper、etcd
AP（高可用）：Eureka、Nacos（AP 模式）
```

**BASE 理论**（AP 模式下的最终一致性）：

| 概念 | 含义 | 示例 |
|------|------|------|
| BA（Basically Available） | 基本可用 | 下单成功但库存延迟更新 |
| S（Soft State） | 软状态 | 允许数据在中间状态停留 |
| E（Eventually Consistency） | 最终一致性 | 经过一段时间后数据达到一致 |

### 1.3 分布式事务解决方案对比

| 方案 | 一致性 | 性能 | 实现复杂度 | 适用场景 |
|------|--------|------|-----------|----------|
| 2PC（两阶段提交） | 强一致 | 低（同步阻塞） | 中 | 传统数据库 |
| Seata AT 模式 | 强一致 | 中 | 低（无侵入） | 业务系统首选 |
| Seata TCC 模式 | 强一致 | 高 | 高（需实现三个方法） | 高性能场景 |
| 本地消息表 | 最终一致 | 高 | 中 | 异步解耦 |
| MQ 事务消息 | 最终一致 | 高 | 中 | 异步场景 |
| Saga 模式 | 最终一致 | 高 | 高（需定义补偿） | 长流程业务 |

## 二、Seata 整体架构

### 2.1 三大组件

```
TM（Transaction Manager）   - 事务管理器，开启/提交/回滚全局事务
RM（Resource Manager）      - 资源管理器，管理分支事务（数据库）
TC（Transaction Coordinator）- 事务协调器（Seata Server），调度全局事务
```

**一次完整的分布式事务流程**：

```
1. TM 开启全局事务 -> TC 生成全局唯一的 XID
2. XID 在调用链中传递（通过 Header/ThreadLocal）
3. RM 注册分支事务到 TC，汇报分支事务状态
4. TM 决定提交或回滚 -> 通知 TC
5. TC 通知所有 RM 提交或回滚分支事务
```

### 2.2 XID 的传播

```
服务 A (TM) -> 开启全局事务 -> 获得 XID: 192.168.1.100:8091:123456
    |
    | Feign 调用，XID 放在 Header 中传递
    v
服务 B (RM) -> 从 Header 获取 XID -> 绑定到当前分支事务
    |
    | Feign 调用
    v
服务 C (RM) -> 从 Header 获取 XID -> 绑定到当前分支事务
```

```java
// Seata 自动通过拦截器传递 XID，无需手动处理
// 底层原理：
// 1. GlobalTransactionalInterceptor 拦截 @GlobalTransactional 方法
// 2. 开启全局事务，获得 XID
// 3. RootContext.bind(xid) 绑定到 ThreadLocal
// 4. Feign 拦截器从 RootContext 获取 XID 放入请求 Header
// 5. 被调用方从 Header 取出 XID 绑定到 RootContext
```

## 三、AT 模式详解（重点）

### 3.1 AT 模式的核心思想

**一句话概括**：AT 模式 = 2PC 的改进版，通过**自动生成 SQL 回滚语句**，让业务代码**零侵入**。

```
传统 2PC 的问题：
  1. 一阶段：协调者询问所有参与者是否可以提交（阻塞等待）
  2. 二阶段：全部同意则提交，否则回滚
  3. 问题：一阶段到二阶段期间资源一直被锁定，性能差

AT 模式的改进：
  1. 一阶段：执行业务 SQL + 自动生成回滚日志（undolog），本地事务直接提交
  2. 二阶段：如果全局提交，异步删除 undo log；如果全局回滚，用 undo log 补偿恢复
  3. 好处：一阶段本地事务直接提交，不阻塞；二阶段靠 undo log 回滚
```

### 3.2 一阶段执行流程（Branch Register + Execute + Report）

```
业务代码执行 SQL：
  UPDATE stock SET count = count - 1 WHERE id = 1

Seata 代理数据源拦截 -> 自动完成以下操作（在一个本地事务中）：

Step 1: 解析 SQL，生成 before image（前置镜像）
  SELECT count FROM stock WHERE id = 1
  beforeImage: { count: 10 }

Step 2: 执行业务 SQL
  UPDATE stock SET count = count - 1 WHERE id = 1

Step 3: 生成 after image（后置镜像）
  SELECT count FROM stock WHERE id = 1
  afterImage: { count: 9 }

Step 4: 插入 undo log 记录
  INSERT INTO undo_log (
    branch_id,
    xid,
    context,                    -- 序列化方式等元信息
    rollback_info,              -- before/after image 的 JSON
    log_status
  ) VALUES (
    12345,
    '192.168.1.100:8091:123456',
    'serializer=jackson',
    '{"beforeImage":{"count":10}, "afterImage":{"count":9}}',
    0
  );

Step 5: 本地事务提交
  COMMIT;

Step 6: 向 TC 注册分支事务，汇报状态
```

**关键点**：Step 1-5 在同一个本地事务中，要么全部成功，要么全部回滚。

### 3.3 二阶段提交（Global Commit）

```
TC 收到 TM 的提交指令 -> 通知所有 RM 提交分支事务

RM 收到提交指令 -> 
  1. 异步删除 undo log（因为已经提交成功，不需要回滚日志了）
  2. 汇报提交完成

DELETE FROM undo_log WHERE xid = '192.168.1.100:8091:123456';
```

**二阶段提交非常快**（异步删除 undo log），不阻塞业务。

### 3.4 二阶段回滚（Global Rollback）

```
TC 收到 TM 的回滚指令 -> 通知所有 RM 回滚分支事务

RM 收到回滚指令 ->
  1. 根据 XID + Branch ID 查找 undo log
  2. 校验 after image 与当前数据是否一致（数据一致性校验）
  3. 生成反向 SQL 并执行（after -> before）
  4. 删除 undo log
  5. 汇报回滚完成
```

**反向 SQL 生成**：

```java
// 原始 SQL（正向操作）
UPDATE stock SET count = count - 1 WHERE id = 1;
// beforeImage: { count: 10 }
// afterImage:  { count: 9  }

// 回滚 SQL（反向操作，用 beforeImage 恢复）
UPDATE stock SET count = 10 WHERE id = 1;
// 用 beforeImage 的值覆盖回去
```

**数据不一致性校验**：

```
Seata 回滚时会检查：
  当前数据库的 count 值 == afterImage 中的 count 值吗？

情况 1：相等 -> 说明没有其他线程修改过，直接回滚
情况 2：不等 -> 说明数据已经被其他事务修改了
  -> Seata 无法自动回滚（怕覆盖别人的修改）
  -> 记录回滚失败，需要人工处理
```

### 3.5 undo_log 表结构

每个使用 AT 模式的数据库都需要创建这张表：

```sql
CREATE TABLE `undo_log` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `branch_id` BIGINT NOT NULL COMMENT '分支事务 ID',
  `xid` VARCHAR(128) NOT NULL COMMENT '全局事务 ID',
  `context` VARCHAR(128) NOT NULL COMMENT '上下文（序列化方式等）',
  `rollback_info` LONGBLOB NOT NULL COMMENT '回滚信息（JSON）',
  `log_status` INT NOT NULL COMMENT '状态（0正常，1全局防御）',
  `log_created` DATETIME NOT NULL,
  `log_modified` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**rollback_info 的结构**：

```json
{
  "@class": "io.seata.rm.datasource.undo.BranchUndoLog",
  "xid": "192.168.1.100:8091:123456",
  "branchId": 12345,
  "sqlUndoLogs": [
    {
      "tableName": "stock",
      "beforeImage": {
        "rows": [
          { "fields": [
            { "name": "id", "type": 4, "value": 1 },
            { "name": "count", "type": 4, "value": 10 }
          ]}
        ]
      },
      "afterImage": {
        "rows": [
          { "fields": [
            { "name": "id", "type": 4, "value": 1 },
            { "name": "count", "type": 4, "value": 9 }
          ]}
        ]
      }
    }
  ]
}
```

### 3.6 全局锁（Global Lock）—— 解决脏写问题

```
场景：
  全局事务 A 修改了 stock.count = 9（本地事务已提交）
  本地事务 B（不受 Seata 管理）也修改了 stock.count = 5
  全局事务 A 二阶段回滚 -> 恢复到 count = 10
  结果：事务 B 的修改被覆盖了（脏写）

Seata 的解决方案：全局锁
```

**全局锁工作机制**：

```
一阶段执行：
  1. RM 执行 SQL 前，先向 TC 申请全局锁（branch_id + 表名 + 主键）
  2. TC 检查是否有其他全局事务持有该行的全局锁
     - 无锁：授予，执行 SQL
     - 有锁：等待（重试），超过重试次数抛异常
  3. SQL 执行成功后，全局锁不释放（直到二阶段）

二阶段提交：
  -> 释放全局锁

二阶段回滚：
  -> 用 undo log 回滚 -> 释放全局锁
```

```
申请全局锁 SQL（伪代码）：
  TC 收到: lock_key = "stock:1"  (表名:主键值)
  检查: 是否有其他 XID 持有 stock:1 的全局锁
  无冲突 -> 注册当前 XID 的锁
  有冲突 -> 等待或拒绝
```

**注意事项**：
- AT 模式下，**同一行数据不能被非 Seata 管理的本地事务同时修改**
- 如果确实需要，需要设置 `lock.retry.times` 和 `lock.retry.interval`

## 四、Seata Server 部署

### 4.1 存储模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| file | 数据存储在本机文件 | 开发测试，不支持集群 |
| db | 数据存储到数据库（MySQL） | 生产环境，支持集群 |
| redis | 数据存储到 Redis（Seata 1.3+） | 高性能场景 |

**db 模式需要的表**：

```sql
-- 全局事务表
CREATE TABLE `global_table` (
  `xid` VARCHAR(128) NOT NULL,
  `transaction_id` BIGINT,
  `status` TINYINT NOT NULL,
  `application_id` VARCHAR(64),
  `transaction_service_group` VARCHAR(64),
  `transaction_name` VARCHAR(128),
  `timeout` INT,
  `begin_time` BIGINT,
  `gmt_modified` DATETIME,
  PRIMARY KEY (`xid`)
);

-- 分支事务表
CREATE TABLE `branch_table` (
  `branch_id` BIGINT NOT NULL,
  `xid` VARCHAR(128) NOT NULL,
  `transaction_id` BIGINT,
  `status` TINYINT,
  `client_id` VARCHAR(64),
  `application_data` VARCHAR(2000),
  `gmt_modified` DATETIME,
  PRIMARY KEY (`branch_id`)
);

-- 全局锁表
CREATE TABLE `lock_table` (
  `row_key` VARCHAR(128) NOT NULL,
  `xid` VARCHAR(128),
  `transaction_id` BIGINT,
  `branch_id` BIGINT,
  `resource_id` VARCHAR(256),
  `table_name` VARCHAR(64),
  `pk` VARCHAR(36),
  `gmt_modified` DATETIME,
  PRIMARY KEY (`row_key`)
);
```

### 4.2 集群部署

```
Seata Server 集群（多节点） + 共享数据库（存储全局事务/分支事务/全局锁）
  ↓
Nacos 注册中心（Seata Server 注册自身）
  ↓
应用客户端通过 Nacos 发现 Seata Server 地址
```

```yaml
# 客户端配置（application.yml）
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: my_test_tx_group  # 事务分组
  service:
    vgroup-mapping:
      my_test_tx_group: default       # 分组映射到集群名
    grouplist:
      default: 127.0.0.1:8091        # Seata Server 地址（配合 Nacos 可省略）
  registry:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
```

## 五、使用方式

### 5.1 最简单用法（@GlobalTransactional）

```java
@Service
public class OrderServiceImpl implements OrderService {
    
    @Autowired
    private OrderClient orderClient;
    @Autowired
    private StockClient stockClient;
    @Autowired
    private AccountClient accountClient;
    
    @GlobalTransactional(name = "createOrder", rollbackFor = Exception.class)
    public void createOrder(OrderDTO dto) {
        // 1. 创建订单（order-service）
        Long orderId = orderClient.create(dto);
        
        // 2. 扣减库存（stock-service）
        stockClient.deduct(dto.getGoodsId(), dto.getCount());
        
        // 3. 扣减余额（account-service）
        accountClient.deduct(dto.getUserId(), dto.getAmount());
        
        // 4. 更新订单状态
        orderClient.updateStatus(orderId, OrderStatus.PAID);
    }
}
```

**只要一个步骤抛异常，所有操作自动回滚**。

### 5.2 数据源代理

Seata 需要代理数据源才能拦截 SQL：

```java
// Seata 1.0+ 自动代理（推荐）
// 引入 seata-spring-boot-starter 后，数据源自动被 DataSourceProxy 包装

// 手动代理（老版本）
@Configuration
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        return new DruidDataSource();
    }
    
    @Bean
    public DataSourceProxy dataSourceProxy(DataSource dataSource) {
        return new DataSourceProxy(dataSource);
    }
}
```

## 六、AT 模式深入理解

### 6.1 为什么 AT 模式比传统 2PC 快

```
传统 2PC：
  一阶段：协调者询问 -> 参与者锁定资源 -> 等待回复 -> 全部锁定
  二阶段：提交/回滚 -> 释放锁
  问题：一阶段到二阶段期间，资源一直被锁定（同步阻塞）

AT 模式：
  一阶段：执行业务 SQL -> 生成 undo log -> 本地事务直接提交 -> 注册分支
  二阶段：异步删除 undo log（提交）或 用 undo log 回滚
  优势：一阶段不锁资源（靠全局锁做并发控制），本地事务立即提交
```

### 6.2 AT 模式的限制

| 限制 | 说明 | 解决方式 |
|------|------|----------|
| 必须有主键 | undo log 需要主键定位记录 | 表设计必须有主键 |
| 不支持 DDL | 表结构变更无法生成 undo log | DDL 不走 Seata |
| 不支持嵌套全局事务 | 一个方法不能有多个 @GlobalTransactional | 拆分为独立事务 |
| 不支持 select for update | 需要加全局锁 | 改用 AT 模式的 lock |
| 大批量操作性能差 | 每条记录都要生成 image | 分批处理或改用 TCC |

### 6.3 AT 模式的并发冲突场景

```
场景一：Seata 全局事务 vs 本地事务
  全局事务 A 正在回滚，本地事务 B 同时修改了同一行
  结果：回滚可能失败（数据不一致性校验不通过）
  解决：全局锁机制保证同一时刻只有一个事务能修改该行

场景二：两个 Seata 全局事务同时修改同一行
  全局事务 A 和 B 同时修改 stock.count
  全局锁保证只有一个事务能先执行
  另一个事务等待或超时失败

场景三：全局事务回滚期间，本地事务正在读
  MVCC 机制保证读操作读到的是回滚前的快照
  MySQL InnoDB 的 ReadView 隔离
```

## 七、TCC 模式（Try-Confirm-Cancel）

### 7.1 TCC 的三个方法

```java
@LocalTCC
public interface StockTCCService {
    
    // Try：预留资源（冻结库存）
    @TwoPhaseBusinessAction(name = "stockDeduct", commitMethod = "confirm", rollbackMethod = "cancel")
    boolean tryDeduct(@BusinessActionContextParameter Long goodsId, 
                      @BusinessActionContextParameter Integer count);
    
    // Confirm：确认执行（扣减库存）
    boolean confirm(BusinessActionContext context);
    
    // Cancel：取消操作（恢复库存）
    boolean cancel(BusinessActionContext context);
}
```

```
Try 阶段：检查并预留资源（不是真正扣减，而是冻结）
  -> 库存表增加 frozen_count，减少 available_count

Confirm 阶段：真正执行业务（用预留的资源）
  -> frozen_count 减少

Cancel 阶段：释放预留资源
  -> frozen_count 减少，available_count 增加
```

### 7.2 AT vs TCC

| 维度 | AT 模式 | TCC 模式 |
|------|---------|----------|
| 侵入性 | 无侵入（自动代理 SQL） | 高侵入（需实现三个方法） |
| 性能 | 中（需要生成 undo log） | 高（无 undo log，自定义逻辑） |
| 一致性 | 强一致（undo log 回滚） | 强一致（业务自己保证） |
| 开发成本 | 低 | 高 |
| 适用场景 | 大多数业务场景 | 高性能要求、非关系型数据库 |

## 八、生产实践与避坑

### 8.1 全局事务超时

```yaml
seata:
  tm:
    default-global-transaction-timeout: 60000  # 全局事务超时 60s
  client:
    rm:
      report-retry-count: 5      # 分支注册重试次数
      table-meta-check-enable: true  # 表结构变更检查
    tm:
      commit-retry-count: 5      # 提交重试次数
      rollback-retry-count: 5    # 回滚重试次数
```

**超时后会怎样**：
- TM 超时 -> 触发回滚
- RM 分支超时 -> 本地事务可能还在执行
- 解决：合理设置超时时间（> 所有分支执行时间之和）

### 8.2 空回滚问题

```
场景：
  Try 阶段因为网络超时，Branch Register 没有到达 TC
  但全局事务超时，TC 通知回滚
  RM 收到回滚指令，但找不到 undo log（Try 没执行）

解决：Seata 1.4+ 自动处理空回滚
  RM 回滚时如果发现没有 undo log，创建一条空回滚记录
  防止后续 Try 到达后产生脏数据
```

### 8.3 防悬挂问题

```
场景：
  Try 请求因为网络延迟，晚于 Cancel 到达
  Cancel 发现没有 undo log，执行了空回滚
  然后 Try 到达，执行了预留资源
  结果：资源被预留但永远不会被确认或取消

解决：空回滚时记录 branchId
  Try 到达时检查 branchId 是否已回滚过
  如果已回滚，拒绝 Try
```

### 8.4 undo_log 清理策略

```
问题：undo_log 表会越来越大

解决方案：
1. 二阶段提交成功后，undo log 会被异步删除
2. 如果二阶段回滚失败，undo log 会残留
3. 定期清理脚本：
   DELETE FROM undo_log WHERE log_created < DATE_SUB(NOW(), INTERVAL 7 DAY);
```

### 8.5 分布式事务的降级策略

```java
// 当 Seata Server 不可用时，降级为本地事务
@GlobalTransactional(fallbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    ...
}

// 配置降级
seata:
  enable-degrade: true  # 开启降级
  disable-global-transaction: false  # 关闭全局事务（降级为 @Transactional）
```
