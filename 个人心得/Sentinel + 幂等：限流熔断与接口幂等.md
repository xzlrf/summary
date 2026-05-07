# Sentinel + 幂等：限流熔断与接口幂等

## 一、Sentinel 流量防卫兵

### 1.1 Sentinel 能做什么

```
Sentinel = 限流 + 熔断 + 降级 + 系统保护 + 热点探测
```

| 功能 | 解决的问题 | 类比 |
|------|-----------|------|
| 流量控制 | 防止瞬时流量打垮服务 | 高速公路收费站限速 |
| 熔断降级 | 下游服务不可用时快速失败，避免级联故障 | 电路保险丝 |
| 系统保护 | 根据系统负载（CPU/Load）自适应限流 | 人体自我保护机制 |
| 热点参数限流 | 对特定参数值限流（如某个商品 ID） | 某件爆款商品限购 |

### 1.2 核心概念：资源、规则、Slot Chain

```
Resource（资源）= 被保护的东西（接口、方法、代码块）
Rule（规则）= 怎么保护（限流阈值、熔断策略）
Slot Chain（插槽链）= Sentinel 内部的处理流水线
```

**Slot Chain 执行流程**：

```
Entry 进入 -> 
  -> NodeSelectorSlot（收集资源路径，构建树形结构）
  -> ClusterBuilderSlot（聚类统计，记录全局指标）
  -> StatisticSlot（核心统计：QPS、线程数、异常数）
  -> FlowSlot（限流检查）
  -> DegradeSlot（熔断降级检查）
  -> SystemSlot（系统保护检查）
  -> AuthoritySlot（黑白名单检查）
  -> 通过 -> 执行目标代码
```

### 1.3 流量控制

#### 1.3.1 限流策略

| 策略 | 原理 | 适用场景 |
|------|------|----------|
| 直接拒绝（Reject） | 超过阈值直接抛 `BlockException` | 常规接口 |
| Warm Up（预热） | 冷启动，阈值从 1/3 逐步增长到设定值 | 秒杀系统启动时 |
| 匀速排队（Rate Limiter） | 漏桶算法，请求匀速通过，超时拒绝 | 处理突发流量，排队消费 |

```java
// 直接拒绝
FlowRule rule = new FlowRule("queryGoods");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // 按 QPS
rule.setCount(100);                          // 阈值 100 QPS
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_REJECT);

// Warm Up 预热
FlowRule warmUpRule = new FlowRule("seckill");
warmUpRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
warmUpRule.setCount(1000);
warmUpRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);
warmUpRule.setWarmUpPeriodSec(10);  // 10秒内从 333 增长到 1000

// 匀速排队
FlowRule rateLimiterRule = new FlowRule("order");
rateLimiterRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rateLimiterRule.setCount(50);
rateLimiterRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
rateLimiterRule.setMaxQueueingTimeMs(500);  // 最大排队等待 500ms
```

#### 1.3.2 流控模式

| 模式 | 含义 | 示例 |
|------|------|------|
| 直接 | 资源本身达到阈值就限流 | 接口 QPS > 100 就限 |
| 关联 | 关联资源达到阈值就限流本资源 | 写接口繁忙时限制读接口 |
| 链路 | 只针对从某个链路过来的请求限流 | 只限制从 Gateway 来的请求 |

**关联模式**（Write 影响 Read）：
```java
// 当 /api/order/write（写订单）QPS 超过 50 时，限制 /api/order/read
FlowRule rule = new FlowRule("/api/order/read");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(200);
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_REJECT);
rule.setRefResource("/api/order/write");  // 关联资源
rule.setStrategy(RuleConstant.STRATEGY_RELATE);  // 关联模式
```

**链路模式**（只限制特定入口）：
```java
// 只对从 Gateway 进入的请求限流，内部调用不受影响
FlowRule rule = new FlowRule("orderService.create");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);
rule.setStrategy(RuleConstant.STRATEGY_CHAIN);
rule.setRefResource("gateway-entry");  // 入口资源名
```

### 1.4 熔断降级

#### 1.4.1 熔断 vs 降级 vs 限流

| 概念 | 方向 | 触发条件 | 类比 |
|------|------|----------|------|
| 限流 | 保护自己 | 自己流量过大 | 收费站限流 |
| 熔断 | 保护自己 | 下游服务不可用 | 电路保险丝跳闸 |
| 降级 | 保护上游 | 自身负载过高或下游不可用 | 飞机抛货物减轻重量 |

**一句话理解**：限流防自己被打死，熔断防别人拖死自己，降级是为了保命主动放弃功能。

#### 1.4.2 熔断策略

| 策略 | 判断指标 | 触发条件 |
|------|----------|----------|
| 慢调用比例 | 响应时间（RT） | 慢调用比例 > 阈值 |
| 异常比例 | 异常数 / 总请求数 | 异常比例 > 阈值 |
| 异常数 | 异常数量 | 异常数 > 阈值 |

```java
// 慢调用比例熔断
DegradeRule slowRule = new DegradeRule("callUserService");
slowRule.setGrade(RuleConstant.DEGRADE_GRADE_SLOW_REQUEST_RATIO);
slowRule.setCount(0.5);              // 慢调用比例超过 50% 触发熔断
slowRule.setSlowRatioThreshold(0.5);
slowRule.setTimeWindow(10);          // 熔断持续 10 秒
slowRule.setMinRequestAmount(5);     // 最小请求数（少于 5 个不统计）
slowRule.setStatIntervalMs(1000);    // 统计时长 1 秒

// 异常比例熔断
DegradeRule errorRule = new DegradeRule("callPaymentService");
errorRule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
errorRule.setCount(0.3);             // 异常比例超过 30%
errorRule.setTimeWindow(10);         // 熔断 10 秒
errorRule.setMinRequestAmount(5);

// 异常数熔断
DegradeRule errorCountRule = new DegradeRule("callLogisticsService");
errorCountRule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_COUNT);
errorCountRule.setCount(10);         // 异常数超过 10 个就熔断
errorCountRule.setTimeWindow(10);    // 熔断 10 秒
errorCountRule.setMinRequestAmount(5);
```

#### 1.4.3 熔断状态机

```
CLOSED（关闭） -> 正常放行，统计异常/慢调用比例
       ↓ 达到熔断阈值
OPEN（打开） -> 所有请求直接拒绝，开始倒计时（TimeWindow）
       ↓ TimeWindow 结束
HALF_OPEN（半开） -> 放行一个请求探测
       ↓ 成功 -> CLOSED（恢复）
       ↓ 失败 -> OPEN（继续熔断）
```

### 1.5 热点参数限流

热点参数限流：**对同一个接口的不同参数值分别限流**。

```java
// 接口：查询商品详情
@GetMapping("/goods/{id}")
@SentinelResource(value = "queryGoods", blockHandler = "handleBlock")
public Result queryGoods(@PathVariable Long id) { ... }

// 热点规则：商品 ID=1 的 QPS 限制为 10，其他商品 ID 默认 100
ParamFlowRule rule = new ParamFlowRule("queryGoods");
rule.setParamIdx(0);                    // 第 0 个参数（id）
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);                     // 默认阈值

// 特殊参数值单独限流
ParamFlowItem item = new ParamFlowItem();
item.setObject("1");                    // 商品 ID = 1（爆款商品）
item.setCount(10);                      // 只允许 10 QPS
item.setClassType(long.class.getName());
rule.setParamFlowItemList(List.of(item));
```

**应用场景**：
- 秒杀场景中，某个热门商品被大量请求访问，单独限制该商品 ID 的 QPS
- 防刷：同一个用户 ID 短时间内大量请求，限制该用户的访问频率

### 1.6 系统保护（Adaptive Protection）

```java
// 当系统负载过高时，自动拒绝新请求
SystemRule rule = new SystemRule();
rule.setHighestSystemLoad(5.0);    // 系统 Load > 5 触发保护
rule.setHighestCpuUsage(0.8);      // CPU 使用率 > 80% 触发
rule.setAvgRt(50);                 // 平均 RT > 50ms 触发
rule.setQps(1000);                 // QPS > 1000 触发
rule.setThreadCount(200);          // 线程数 > 200 触发
```

系统保护是**最后一道防线**，当上述所有规则都没拦住，但系统已经快撑不住了，系统保护自适应地拒绝请求。

### 1.7 与 OpenFeign 整合

```java
// 方式一：@FeignClient + Sentinel 自动生效（需开启配置）
@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient {
    @GetMapping("/api/users/{id}")
    Result<User> getUserById(@PathVariable Long id);
}

// 配置开启
feign:
  sentinel:
    enabled: true

// 方式二：手动 @SentinelResource 包裹 Feign 调用
@Service
public class OrderServiceImpl implements OrderService {
    @Autowired
    private UserClient userClient;
    
    @SentinelResource(value = "getUserFromService", 
        fallback = "getUserFallback",
        blockHandler = "getUserBlockHandler")
    public User getUser(Long id) {
        return userClient.getUserById(id).getData();
    }
    
    // fallback：业务降级（服务异常时调用）
    public User getUserFallback(Long id, Throwable e) {
        return new User(id, "默认用户");
    }
    
    // blockHandler：被 Sentinel 限流/熔断时调用
    public User getUserBlockHandler(Long id, BlockException e) {
        throw new BizException("服务被限流/熔断");
    }
}
```

**fallback vs blockHandler 的区别**：

| | fallback | blockHandler |
|---|----------|-------------|
| 触发原因 | 业务异常（下游服务抛异常） | Sentinel 限流/熔断（BlockException） |
| 方法签名 | 与原方法相同 + `Throwable` 参数 | 与原方法相同 + `BlockException` 参数 |
| 返回类型 | 必须与原方法相同 | 必须与原方法相同 |
| 用途 | 服务降级，返回兜底数据 | 限流/熔断提示 |

### 1.8 Sentinel Dashboard 与 Nacos 持久化

**问题**：Sentinel Dashboard 配置的规则默认存储在内存中，重启后丢失。

**解决方案**：将规则持久化到 Nacos。

```yaml
# 应用侧配置
spring:
  cloud:
    sentinel:
      datasource:
        flow:  # 流控规则
          nacos:
            server-addr: ${NACOS_ADDR}
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow
        degrade:  # 降级规则
          nacos:
            server-addr: ${NACOS_ADDR}
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            rule-type: degrade
```

```
规则同步流程：
Dashboard 修改规则 -> 写入 Nacos -> 
  应用侧监听 Nacos 配置变化 -> 加载新规则到内存
应用侧规则变化 -> 推送至 Nacos -> Dashboard 读取展示
```

## 二、接口幂等性

### 2.1 什么是幂等

**幂等性（Idempotency）**：同一个操作，执行一次和执行多次，结果相同。

```
GET /users/1       幂等（查多少次都一样）
DELETE /users/1    幂等（删多少次都是删除）
PUT /users/1       幂等（覆盖更新，结果一致）
POST /users        非幂等（每次创建新用户）
```

**为什么需要保证幂等**：
1. **网络重试**：RPC 调用超时，调用方重试导致重复请求
2. **前端重复提交**：用户连续点击"提交"按钮
3. **消息队列消费**：消息重复投递
4. **支付回调**：第三方支付重复通知

### 2.2 Token 机制（防重放）

**思路**：请求前先获取一个一次性 Token，提交时携带 Token，服务端校验并删除。

```
前端 -> [获取 Token] -> 服务端生成 Token 存入 Redis -> 返回 Token
前端 -> [提交请求 + Token] -> 服务端校验 Token（DEL 原子操作）-> 
  -> Token 存在：执行业务
  -> Token 不存在：拒绝（重复提交）
```

```java
// 1. 生成 Token
@GetMapping("/token")
public Result<String> getToken() {
    String token = UUID.randomUUID().toString().replace("-", "");
    // 存入 Redis，设置过期时间 5 分钟
    redisTemplate.opsForValue().set("idempotent:token:" + token, "1", 5, TimeUnit.MINUTES);
    return Result.success(token);
}

// 2. 校验 Token（Lua 脚本保证原子性）
@PostMapping("/order")
public Result createOrder(@RequestHeader("X-Idempotent-Token") String token, 
                           @RequestBody OrderDTO dto) {
    // Lua 脚本：检查 + 删除，原子操作
    String lua = "if redis.call('get', KEYS[1]) == '1' then " +
                 "  redis.call('del', KEYS[1]) " +
                 "  return 1 " +
                 "else " +
                 "  return 0 " +
                 "end";
    
    Long result = redisTemplate.execute(
        new DefaultRedisScript<>(lua, Long.class),
        List.of("idempotent:token:" + token)
    );
    
    if (result == 0) {
        return Result.fail("请勿重复提交");
    }
    
    // 执行业务逻辑
    orderService.create(dto);
    return Result.success();
}
```

**为什么用 Lua 而不是先 get 再 del**：
```
// 非原子操作的问题：
if (redis.get(key) != null) {    // 线程 A 和 B 同时 get 到 token
    redis.del(key);               // A 和 B 都删除成功，都执行业务 -> 重复
}

// Lua 脚本原子执行：
// GET + DEL 在 Redis 中是单线程执行的，不存在并发问题
```

### 2.3 数据库唯一索引

**思路**：利用数据库唯一约束，重复插入会报 `DuplicateKeyException`。

```java
// 订单表增加唯一索引
// CREATE UNIQUE INDEX uk_order_no ON orders(order_no);

@PostMapping("/order")
public Result createOrder(@RequestBody OrderDTO dto) {
    Order order = new Order();
    order.setOrderNo(dto.getOrderNo());  // 前端生成的唯一订单号
    order.setUserId(dto.getUserId());
    order.setAmount(dto.getAmount());
    
    try {
        orderMapper.insert(order);
        return Result.success();
    } catch (DuplicateKeyException e) {
        return Result.fail("订单已存在，请勿重复提交");
    }
}
```

**适用场景**：
- 订单创建、支付记录插入等需要持久化的场景
- 前端生成唯一业务编号（如 `ORDER_用户ID_时间戳_随机数`）

**注意事项**：
- 唯一索引冲突会抛异常，需要捕获并友好提示
- 在事务中使用，异常会回滚，不影响数据一致性
- 配合 `INSERT ... ON DUPLICATE KEY UPDATE` 可实现幂等更新

### 2.4 去重表

```java
// 去重表结构
// CREATE TABLE idempotent_record (
//     id BIGINT PRIMARY KEY AUTO_INCREMENT,
//     biz_type VARCHAR(32) NOT NULL,    -- 业务类型
//     biz_no VARCHAR(64) NOT NULL,      -- 业务唯一编号
//     status TINYINT DEFAULT 0,         -- 处理状态
//     create_time DATETIME DEFAULT NOW(),
//     UNIQUE KEY uk_biz (biz_type, biz_no)
// ) ENGINE=InnoDB;

@Transactional
public void processPayment(String orderNo, BigDecimal amount) {
    // 1. 插入去重表（唯一约束保证幂等）
    IdempotentRecord record = new IdempotentRecord();
    record.setBizType("PAYMENT");
    record.setBizNo(orderNo);
    
    int rows = idempotentMapper.insert(record);
    if (rows == 0) {
        // 唯一冲突，说明已处理过
        log.warn("重复请求，orderNo={}", orderNo);
        return;
    }
    
    // 2. 执行实际业务
    paymentService.doPay(orderNo, amount);
    
    // 3. 更新状态
    idempotentMapper.updateStatus(orderNo, 1);
}
```

**去重表 vs 唯一索引**：
- 去重表独立于业务表，不污染业务数据模型
- 可以记录处理状态、重试次数、处理结果等元数据
- 适合 MQ 消费场景（消息 ID 作为 bizNo）

### 2.5 分布式锁实现幂等

```java
@PostMapping("/order")
public Result createOrder(@RequestBody OrderDTO dto) {
    String lockKey = "order:create:" + dto.getUserId();
    RLock lock = redissonClient.getLock(lockKey);
    
    try {
        // 尝试获取锁，最多等待 3 秒，锁自动释放时间 10 秒
        if (!lock.tryLock(3, 10, TimeUnit.SECONDS)) {
            return Result.fail("操作过于频繁，请稍后重试");
        }
        
        // 检查是否已存在订单
        Order exist = orderMapper.selectByOrderNo(dto.getOrderNo());
        if (exist != null) {
            return Result.success("订单已存在");
        }
        
        // 创建订单
        orderService.create(dto);
        return Result.success();
    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

**适用场景**：需要"先查询再插入"的幂等场景（防止并发插入）。

### 2.6 各种方案对比

| 方案 | 性能 | 可靠性 | 适用场景 | 局限性 |
|------|------|--------|----------|--------|
| Token 机制 | 高（Redis 操作） | 高（Lua 原子） | 表单提交、前端防重 | 需要额外获取 Token 的接口 |
| 唯一索引 | 中（DB 操作） | 高（DB 保证） | 订单创建、支付记录 | 依赖数据库，高并发时 DB 压力大 |
| 去重表 | 中（DB 操作） | 高（DB 保证） | MQ 消费、异步任务 | 需要维护额外表 |
| 分布式锁 | 中（Redis 操作） | 高（Redisson 保证） | 先查后写场景 | 加锁有性能开销 |
| 状态机 | 高 | 中 | 订单状态流转 | 只能保证状态不逆向流转 |

### 2.7 状态机幂等

订单状态流转天然适合幂等：**只允许正向流转，相同状态的重复操作不生效**。

```java
public boolean updateOrderStatus(Long orderId, OrderStatus from, OrderStatus to) {
    int rows = orderMapper.updateStatus(orderId, from, to);
    // UPDATE orders SET status = #{to} 
    // WHERE id = #{orderId} AND status = #{from}
    // rows = 0 说明当前状态不是 from（已被处理或状态已变更）
    return rows > 0;
}

// 使用
boolean success = updateOrderStatus(orderId, PAID, SHIPPED);
if (!success) {
    log.info("订单状态已变更，跳过处理, orderId={}", orderId);
}
```

### 2.8 MQ 消费幂等

```java
@RabbitListener(queues = "order.queue")
public void handleOrderMessage(Message message, Channel channel) {
    String msgId = message.getMessageProperties().getMessageId();
    String bizNo = extractBizNo(message);
    
    // 方案一：用消息 ID 做去重
    String key = "mq:dedup:" + msgId;
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(key, "1", 24, TimeUnit.HOURS);
    
    if (Boolean.FALSE.equals(success)) {
        // 消息已处理，直接 ACK
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        return;
    }
    
    try {
        // 处理消息
        processOrder(bizNo);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        // 处理失败，NACK 重试
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
        throw e;
    }
}
```

## 三、Sentinel + 幂等 生产实践

### 3.1 接口防重复提交的完整方案

```
前端：提交后禁用按钮 + 路由拦截（防用户操作）
  ↓
网关层：Token 校验（防重放攻击）
  ↓
应用层：分布式锁（防并发提交）
  ↓
数据层：唯一索引（最后一道防线）
```

### 3.2 Sentinel 限流阈值怎么定

```
1. 压测：用 JMeter/Wrk 逐步增加并发，找到系统的最大 QPS
2. 除以安全系数：阈值 = 压测最大 QPS * 0.8（留 20% 余量）
3. 观察生产指标：根据实际 CPU/内存/RT 调整
4. 分级设置：
   - 核心接口（下单/支付）：保守（阈值偏低）
   - 普通接口（查询）：宽松
   - 非核心接口（日志/统计）：可以限掉
```

### 3.3 被限流后怎么给前端反馈

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BlockException.class)
    public Result handleBlockException(BlockException e) {
        if (e instanceof FlowException) {
            return Result.fail("系统繁忙，请稍后重试");
        }
        if (e instanceof DegradeException) {
            return Result.fail("服务降级中，请稍后重试");
        }
        return Result.fail("请求被拦截");
    }
}
```

### 3.4 缓存 + 降级 组合拳

```java
@SentinelResource(value = "getGoodsDetail",
    fallback = "getGoodsDetailFallback")
public GoodsDTO getGoodsDetail(Long id) {
    // 1. 先查缓存
    GoodsDTO cached = cacheManager.get("goods:" + id);
    if (cached != null) {
        return cached;
    }
    
    // 2. 查数据库
    GoodsDTO goods = goodsMapper.selectById(id);
    cacheManager.put("goods:" + id, goods, 30, TimeUnit.MINUTES);
    return goods;
}

// 降级方法：返回缓存中的旧数据
public GoodsDTO getGoodsDetailFallback(Long id, Throwable e) {
    // 尝试从缓存获取旧数据
    return cacheManager.get("goods:" + id);
}
```
