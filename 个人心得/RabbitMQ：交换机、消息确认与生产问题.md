# RabbitMQ：交换机、消息确认、死信队列与生产问题

## 一、RabbitMQ 整体架构

### 1.1 核心概念

```
Producer（生产者）-> 发送消息 -> Exchange（交换机）-> 路由规则 -> Queue（队列）-> Consumer（消费者）

Key 组件：
  Exchange：接收消息，按规则路由到队列
  Binding：交换机和队列之间的绑定规则
  RoutingKey：路由键，决定消息去哪
  VirtualHost：虚拟主机，隔离不同环境的队列（dev/test/prod）
```

### 1.2 为什么用 MQ

| 场景 | 说明 | 示例 |
|------|------|------|
| 异步解耦 | 生产者不需要等消费者处理完 | 下单后发短信，不用等短信发完 |
| 流量削峰 | 缓冲瞬时高并发 | 秒杀场景把请求先塞入 MQ |
| 数据同步 | 变更事件广播给多个消费者 | 订单状态变更通知仓储/财务/物流 |

## 二、交换机类型（重点）

### 2.1 四种交换机对比

| 类型 | 路由规则 | 适用场景 | 示例 |
|------|----------|----------|------|
| Direct | RoutingKey 精确匹配 | 单播、点对点 | 日志级别 routing: error -> error-queue |
| Fanout | 广播，忽略 RoutingKey | 一个消息多个消费者 | 订单创建后同时通知短信/邮件/积分 |
| Topic | RoutingKey 模式匹配（* 和 #） | 灵活路由 | user.# 匹配 user.create、user.delete |
| Headers | 根据消息 Header 匹配 | 少用 | 根据自定义头字段路由 |

### 2.2 Direct Exchange（直连交换机）

```
Producer -> Exchange(direct) + routingKey="order.create"
  -> Binding: queue_order 绑定 routingKey="order.create"
  -> 消息只到 queue_order
```

```java
// 声明交换机
@Bean
public DirectExchange directExchange() {
    return new DirectExchange("order.direct");
}

// 声明队列
@Bean
public Queue orderQueue() {
    return new Queue("queue.order");
}

// 绑定：routingKey = "order.create"
@Bean
public Binding orderBinding(Queue orderQueue, DirectExchange directExchange) {
    return BindingBuilder.bind(orderQueue)
        .to(directExchange)
        .with("order.create");
}
```

### 2.3 Fanout Exchange（扇出交换机）

```
Producer -> Exchange(fanout)
  -> Binding: queue_sms, queue_email, queue_points 全部绑定
  -> 消息同时到三个队列（广播）
```

```java
@Bean
public FanoutExchange fanoutExchange() {
    return new FanoutExchange("order.fanout");
}

// 不需要 routingKey，绑定的队列都会收到
@Bean
public Binding smsBinding(Queue smsQueue, FanoutExchange fanoutExchange) {
    return BindingBuilder.bind(smsQueue).to(fanoutExchange);
}
```

### 2.4 Topic Exchange（主题交换机）

**通配符规则**：
- `*`：匹配一个单词（以 `.` 分隔）
- `#`：匹配零个或多个单词

```
routingKey 格式：模块.操作.类型

绑定规则：
  user.#    -> 匹配 user.create、user.delete、user.create.vip
  user.*    -> 匹配 user.create，但不匹配 user.create.vip
  *.create  -> 匹配 user.create、order.create
  #         -> 匹配所有

示例：
  routingKey = "order.create.success"
  绑定了 "order.#" 的队列 -> 收到
  绑定了 "order.create.*" 的队列 -> 收到
  绑定了 "*.create" 的队列 -> 收不到（* 只匹配一个词）
```

```java
@Bean
public TopicExchange topicExchange() {
    return new TopicExchange("order.topic");
}

@Bean
public Binding topicBinding1(Queue queue1, TopicExchange topicExchange) {
    return BindingBuilder.bind(queue1).to(topicExchange).with("order.#");
}

@Bean
public Binding topicBinding2(Queue queue2, TopicExchange topicExchange) {
    return BindingBuilder.bind(queue2).to(topicExchange).with("*.success");
}
```

## 三、消息确认机制

### 3.1 生产者确认（Publisher Confirm）

**问题**：生产者把消息发出去后，怎么确认 MQ 收到了？

```
流程：
  Producer -> 发送消息 -> RabbitMQ 收到 -> 返回 ACK（成功）或 NACK（失败）
```

```java
// 配置开启
spring.rabbitmq.publisher-confirm-type=correlated

// 回调处理
@Configuration
public class RabbitConfig implements RabbitTemplate.ConfirmCallback {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback(this);
    }
    
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        if (ack) {
            log.info("消息发送成功, id={}", correlationData.getId());
        } else {
            log.error("消息发送失败, id={}, cause={}", correlationData.getId(), cause);
            // 重试或记录数据库
        }
    }
}
```

### 3.2 生产者 Return 回调（消息未路由到队列）

**问题**：消息到了 Exchange，但没有匹配的队列（routingKey 不匹配），消息会丢失。

```java
// 配置
spring.rabbitmq.publisher-returns=true
spring.rabbitmq.template.mandatory=true

// 回调
@Configuration
public class RabbitConfig implements RabbitTemplate.ReturnCallback {
    
    @PostConstruct
    public void init() {
        rabbitTemplate.setReturnCallback(this);
    }
    
    @Override
    public void returnedMessage(Message message, int replyCode, String replyText,
                                 String exchange, String routingKey) {
        log.error("消息未路由到队列: exchange={}, routingKey={}, body={}",
            exchange, routingKey, new String(message.getBody()));
        // 记录日志/告警/重试
    }
}
```

### 3.3 消费者确认（Manual Ack）

**三种确认模式**：

| 模式 | 说明 | 风险 |
|------|------|------|
| auto（自动确认） | 消息一到消费者就自动 ACK | 消费者处理失败消息会丢失 |
| manual（手动确认） | 消费者处理完后手动 ACK | 安全，推荐生产使用 |
| none | 不确认 | 不用 |

```java
// 配置开启手动确认
spring.rabbitmq.listener.simple.acknowledge-mode=manual

// 消费者手动 ACK
@RabbitListener(queues = "queue.order")
public void handleOrder(Message message, Channel channel) throws IOException {
    try {
        String body = new String(message.getBody());
        // 处理业务
        processOrder(body);
        
        // 业务成功 -> 手动 ACK
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        
    } catch (Exception e) {
        log.error("消息处理失败", e);
        // 业务失败 -> NACK，requeue=false 不重回队列（走死信）
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
    }
}
```

**basicReject vs basicNack**：
- `basicReject`：只能拒绝单条消息
- `basicNack`：可以批量拒绝（multiple=true）

### 3.4 完整的消息可靠性保障链路

```
Producer:
  1. Publisher Confirm 确认消息到达 Exchange
  2. Return Callback 确认消息路由到 Queue
  3. 消息落数据库（消息表），定时任务补偿

Broker:
  1. 队列持久化（durable=true）
  2. 消息持久化（deliveryMode=2）

Consumer:
  1. 手动 ACK
  2. 处理失败走死信队列
  3. 幂等消费（防重复）
```

## 四、死信队列（DLQ）

### 4.1 消息什么时候变成死信

| 条件 | 说明 |
|------|------|
| 消息被拒绝（basicNack/basicReject）且 requeue=false | 消费者处理失败 |
| 消息过期（TTL） | 设置了过期时间但没被消费 |
| 队列满了 | 队列达到最大长度 |

### 4.2 死信队列配置

```
正常队列 -> 消息变成死信 -> 转发到死信交换机 -> 路由到死信队列 -> 死信消费者处理
```

```java
// 1. 声明死信交换机和死信队列
@Bean
public DirectExchange dlxExchange() {
    return new DirectExchange("dlx.exchange");
}

@Bean
public Queue dlxQueue() {
    return new Queue("dlx.queue", true);
}

@Bean
public Binding dlxBinding(Queue dlxQueue, DirectExchange dlxExchange) {
    return BindingBuilder.bind(dlxQueue).to(dlxExchange).with("dlx");
}

// 2. 正常队列绑定死信交换机
@Bean
public Queue normalQueue() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-dead-letter-exchange", "dlx.exchange");  // 死信交换机
    args.put("x-dead-letter-routing-key", "dlx");        // 死信 routingKey
    return new Queue("queue.normal", true, false, false, args);
}
```

**死信队列的用途**：
- 记录失败消息，后续人工排查
- 配合定时任务重试（如延迟 5 分钟后重新发送）
- 告警通知（死信数量超过阈值发邮件/短信）

## 五、延迟队列

### 5.1 场景

| 场景 | 说明 |
|------|------|
| 订单超时取消 | 下单 30 分钟后如果未支付自动取消 |
| 延迟重试 | 消息处理失败，延迟 5 分钟重试 |
| 定时任务 | 每天固定时间执行 |

### 5.2 实现方式一：TTL + 死信队列

```
消息设置 TTL -> 正常队列（不消费）-> TTL 到期 -> 死信交换机 -> 死信队列（消费者处理）
```

```java
// 延迟队列（绑死信交换机，但正常消费者不消费，等 TTL 后转死信）
@Bean
public Queue delayQueue() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-dead-letter-exchange", "dlx.exchange");
    args.put("x-dead-letter-routing-key", "dlx");
    args.put("x-message-ttl", 30 * 60 * 1000);  // 30 分钟过期
    return new Queue("queue.delay", true, false, false, args);
}
```

**问题**：同一个队列只能设置一个 TTL，不同延迟时间需要不同队列。

### 5.3 实现方式二：延迟交换机插件（推荐）

```java
// 安装 rabbitmq-delayed-message-exchange 插件后
@Bean
public CustomExchange delayedExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");
    return new CustomExchange("delayed.exchange", "x-delayed-message", true, false, args);
}

// 发送延迟消息
rabbitTemplate.convertAndSend("delayed.exchange", "routing.key", message, msg -> {
    msg.getMessageProperties().setDelay(30 * 60 * 1000);  // 延迟 30 分钟
    return msg;
});
```

## 六、生产三大问题

### 6.1 消息丢失

**可能丢失的环节**：

| 环节 | 原因 | 解决方案 |
|------|------|----------|
| 生产者 -> MQ | 网络异常、MQ 宕机 | Publisher Confirm + 消息落库 |
| MQ 内部 | 队列未持久化、MQ 重启 | 队列持久化 + 消息持久化 + 集群 |
| MQ -> 消费者 | 自动 ACK 后消费者宕机 | 手动 ACK |

**完整保障**：

```
1. 生产者：
   - 开启 Publisher Confirm
   - 消息写本地消息表，定时任务扫描未确认消息重发

2. MQ Broker：
   - 队列 durable=true
   - 消息 deliveryMode=2（持久化）
   - 集群部署 + 镜像队列

3. 消费者：
   - 手动 ACK
   - 先处理业务，再 ACK
   - 失败走死信队列
```

### 6.2 消息重复

**原因**：
- 生产者重发（网络超时导致 Producer 以为没发送成功）
- 消费者 ACK 丢失（MQ 没收到 ACK，重新投递）

**解决方案：消费者幂等**

```java
// 方式一：消息 ID 去重（Redis）
@RabbitListener(queues = "queue.order")
public void handleOrder(Message message, Channel channel) throws IOException {
    String msgId = message.getMessageProperties().getMessageId();
    String key = "mq:dedup:" + msgId;
    
    Boolean success = redisTemplate.opsForValue().setIfAbsent(key, "1", 24, TimeUnit.HOURS);
    if (Boolean.FALSE.equals(success)) {
        log.info("重复消息，跳过, msgId={}", msgId);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        return;
    }
    
    try {
        processOrder(message);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
        throw e;
    }
}

// 方式二：业务唯一键去重（数据库唯一索引）
// 比如订单号作为唯一索引，重复插入会失败
```

### 6.3 消息积压

**现象**：队列中消息越来越多，消费者处理不过来。

**排查思路**：

```
1. 看消费者是否在线（可能挂了）
2. 看消费者处理速度（是不是变慢了）
3. 查消费者日志（是不是有异常一直在重试）
4. 查下游依赖（DB 慢、下游服务不可用）
```

**应急处理**：

```
方案一：临时扩容消费者
  启动更多消费者实例，加速消费

方案二：紧急降级
  消费者只做核心逻辑，非核心操作（日志、通知）先跳过

方案三：消息丢弃（极端情况）
  如果积压消息已经没用了，可以清空队列

方案四：转移积压消息
  新建一个消费者，把积压消息转到新队列，慢慢处理
```

**预防**：
- 监控队列长度（超过阈值告警）
- 消费者设置合理的并发数（`spring.rabbitmq.listener.simple.concurrency`）
- 消费者处理逻辑尽量异步化
