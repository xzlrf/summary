# SpringBoot + Nacos：自动装配、配置加载与服务治理

> 从源码到实战，彻底搞懂 SpringBoot 与 Nacos 的整合

## 目录

- [一、SpringBoot 自动装配](#一springboot-自动装配)
  - [1.1 @SpringBootApplication 拆解](#11-springbootapplication-拆解)
  - [1.2 @EnableAutoConfiguration 原理](#12-enableautoconfiguration-原理)
  - [1.3 spring.factories 与 SPI 机制](#13-springfactories-与-spi-机制)
  - [1.4 自动装配流程全景图](#14-自动装配流程全景图)
  - [1.5 自定义 Starter](#15-自定义-starter)
- [二、配置加载优先级](#二配置加载优先级)
  - [2.1 配置文件优先级](#21-配置文件优先级)
  - [2.2 @Value 与 @ConfigurationProperties](#22-value-与-configurationproperties)
  - [2.3 多环境切换](#23-多环境切换)
  - [2.4 配置加载完整顺序](#24-配置加载完整顺序)
- [三、Nacos 服务注册与发现](#三nacos-服务注册与发现)
  - [3.1 为什么需要服务注册中心](#31-为什么需要服务注册中心)
  - [3.2 Nacos 核心概念](#32-nacos-核心概念)
  - [3.3 服务注册流程](#33-服务注册流程)
  - [3.4 服务发现流程](#34-服务发现流程)
  - [3.5 健康检查机制](#35-健康检查机制)
  - [3.6 服务下线与临时实例](#36-服务下线与临时实例)
- [四、Nacos 配置中心](#四nacos-配置中心)
  - [4.1 配置中心解决什么问题](#41-配置中心解决什么问题)
  - [4.2 集成 Nacos Config](#42-集成-nacos-config)
  - [4.3 动态刷新原理](#43-动态刷新原理)
  - [4.4 @RefreshScope 机制](#44-refreshscope-机制)
  - [4.5 配置监听底层实现](#45-配置监听底层实现)
- [五、命名空间与分组](#五命名空间与分组)
  - [5.1 命名空间 Namespace](#51-命名空间-namespace)
  - [5.2 分组 Group](#52-分组-group)
  - [5.3 Data ID 规范](#53-data-id-规范)
  - [5.4 三者关系与隔离级别](#54-三者关系与隔离级别)
- [六、生产实践](#六生产实践)

---

## 一、SpringBoot 自动装配

### 1.1 @SpringBootApplication 拆解

`@SpringBootApplication` 是一个组合注解，实际由三个注解构成：

```java
@SpringBootConfiguration   // = @Configuration，标识当前类是配置类
@EnableAutoConfiguration   // 开启自动装配（核心）
@ComponentScan             // 组件扫描，默认扫当前包及子包
public @interface SpringBootApplication {
    // ...
}
```

```
@SpringBootApplication
    │
    ├─ @SpringBootConfiguration
    │     └─ @Configuration（标识配置类）
    │
    ├─ @EnableAutoConfiguration  ← 自动装配的核心
    │     └─ @Import(AutoConfigurationImportSelector.class)
    │
    └─ @ComponentScan
          └─ 扫描当前包及子包下的 @Component、@Service 等
```

### 1.2 @EnableAutoConfiguration 原理

关键在 `@Import(AutoConfigurationImportSelector.class)`。

`AutoConfigurationImportSelector` 实现了 `ImportSelector` 接口，在 Spring 容器初始化时会调用其 `selectImports()` 方法：

```java
public class AutoConfigurationImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 1. 获取所有自动装配的配置类
        AutoConfigurationEntry entry = getAutoConfigurationEntry(annotationMetadata);
        return entry.getConfigurations();
    }

    protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        // 2. 加载候选的配置类
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
        // 3. 去重
        configurations = removeDuplicates(configurations);
        // 4. 排除 @SpringBootApplication(exclude=xxx) 指定的类
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        configurations.removeAll(exclusions);
        // 5. 过滤：检查 @Conditional 条件，不满足的剔除
        configurations = getConfigurationClassFilter().filter(configurations);
        // ...
        return new AutoConfigurationEntry(configurations, exclusions);
    }
}
```

核心流程就四步：

```
① 加载 spring.factories / org.springframework.boot.autoconfigure.AutoConfiguration.imports
② 去重
③ 排除用户 exclude 指定的类
④ 按 @Conditional 条件过滤，符合条件的才注册
```

### 1.3 spring.factories 与 SPI 机制

**Java SPI（Service Provider Interface）** 是一种服务发现机制：

```
Java SPI:
  ┌──────────────────────────────────────┐
  │  META-INF/services/                  │
  │    com.xxx.MyInterface               │  ← 文件内容是实现类的全限定名
  └──────────────────────────────────────┘

SpringBoot 扩展了这个机制:
  ┌──────────────────────────────────────┐
  │  META-INF/spring.factories           │  ← SpringBoot 2.7 之前
  │  META-INF/spring/                    │
  │    org.springframework.boot.         │
  │    autoconfigure.AutoConfiguration.  │  ← SpringBoot 2.7+ 新方式
  │    imports                           │
  └──────────────────────────────────────┘
```

**spring.factories 示例：**

```properties
# spring-boot-autoconfigure 包里的 spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
...
```

**SpringBoot 2.7+ 改用新文件：**

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
...
```

**为什么需要 @Conditional？**

自动装配的配置类有上百个，不可能全部生效。`@Conditional` 就是"条件开关"：

```java
@Configuration
@ConditionalOnClass(DataSource.class)        // classpath 有 DataSource 类才生效
@ConditionalOnProperty(prefix = "spring.datasource", name = "url")  // 配置了 url 才生效
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource() {
        // ...
    }
}
```

常见 @Conditional 注解：

| 注解 | 条件 |
|------|------|
| `@ConditionalOnClass` | classpath 存在指定类 |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean |
| `@ConditionalOnProperty` | 配置文件存在指定属性 |
| `@ConditionalOnWebApplication` | 是 Web 应用 |
| `@ConditionalOnMissingClass` | classpath 不存在指定类 |

### 1.4 自动装配流程全景图

```
启动应用
  │
  ▼
@SpringBootApplication
  │
  ├─ @ComponentScan → 扫描当前包及子包，注册 @Component 等
  │
  └─ @EnableAutoConfiguration
        │
        ▼
     AutoConfigurationImportSelector.selectImports()
        │
        ├─ ① SpringFactoriesLoader.loadFactoryNames()
        │     └─ 读取所有 jar 包里的 spring.factories / AutoConfiguration.imports
        │     └─ 得到 100+ 个候选配置类
        │
        ├─ ② 去重、排除 exclude 指定的类
        │
        ├─ ③ @Conditional 过滤
        │     └─ 检查每个配置类的 @ConditionalOnXxx 条件
        │     └─ 不满足的剔除，最终剩下 20~30 个
        │
        └─ ④ 注册到 Spring 容器
              └─ 这些配置类里的 @Bean 被实例化
              └─ 自动装配完成
```

### 1.5 自定义 Starter

自己写一个 Starter 只需要三步：

**第一步：写自动配置类**

```java
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "my.starter", name = "enabled", havingValue = "true", matchIfMissing = true)
    public MyService myService(MyProperties properties) {
        return new MyService(properties.getName());
    }
}
```

**第二步：写配置属性类**

```java
@ConfigurationProperties(prefix = "my.starter")
public class MyProperties {
    private String name = "default";
    // getter / setter
}
```

**第三步：注册 spring.factories**

```
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

或者 SpringBoot 2.7+ 方式：

```
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyAutoConfiguration
```

打成 jar 包后，其他项目引入依赖即可自动装配。

---

## 二、配置加载优先级

### 2.1 配置文件优先级

SpringBoot 会从多个位置加载 `application.yml`，优先级从高到低：

```
① file:./config/              （当前目录下的 config 目录）
② file:./                     （当前目录）
③ classpath:/config/          （类路径下的 config 目录）
④ classpath:/                 （类路径根目录）
```

**同级目录下，不同格式文件的优先级**（从高到低）：

```
.properties > .yaml > .yml
```

**命令行参数 > 配置文件**：

```bash
# 命令行参数优先级最高
java -jar app.jar --server.port=8081
```

完整优先级（从低到高）：

```
1.  @SpringBootApplication 默认属性
2.  classpath: application.yml
3.  classpath: application-{profile}.yml
4.  file:./ application.yml
5.  file:./ application-{profile}.yml
6.  file:./config/ application.yml
7.  file:./config/ application-{profile}.yml
8.  环境变量（JAVA_OPTS、系统环境变量）
9.  命令行参数（--server.port=8081）
10. 测试注解 @TestPropertySource
11. @SpringBootTest(properties = ...)
```

**核心规则：高优先级的配置会覆盖低优先级的同名属性。**

### 2.2 @Value 与 @ConfigurationProperties

| 特性 | @Value | @ConfigurationProperties |
|------|--------|--------------------------|
| 用途 | 单个属性注入 | 批量属性绑定到对象 |
| 松散绑定 | 不支持 | 支持（`user-name` ↔ `userName`） |
| JSR303 校验 | 不支持 | 支持（@Validated） |
| SpEL 表达式 | 支持 | 不支持 |
| 复杂类型 | 不支持 | 支持（List、Map、嵌套对象） |
| 推荐场景 | 注入一两个值 | 配置项较多时 |

```java
// @Value：适合少量配置
@Value("${server.port}")
private int port;

@Value("${my.custom.value:default}")  // 带默认值
private String value;

// @ConfigurationProperties：适合批量配置
@Component
@ConfigurationProperties(prefix = "app")
@Data
public class AppProperties {
    private String name;
    private int port;
    private List<String> urls;
    private Map<String, String> headers;
    private Database database;  // 嵌套对象

    @Data
    public static class Database {
        private String url;
        private String username;
    }
}
```

**松散绑定示例：**

```yaml
app:
  user-name: admin       # 中划线
  user_name: admin       # 下划线
  userName: admin        # 驼峰
  USER_NAME: admin       # 大写
# 以上四种写法都能绑定到 userName 字段
```

### 2.3 多环境切换

**方式一：多 profile 文件**

```
application.yml              # 公共配置
application-dev.yml          # 开发环境
application-test.yml         # 测试环境
application-prod.yml         # 生产环境
```

```yaml
# application.yml
spring:
  profiles:
    active: dev              # 指定激活的环境
```

**方式二：YAML 多文档块（一个文件搞定）**

```yaml
# application.yml
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081

---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

**方式三：命令行指定**

```bash
java -jar app.jar --spring.profiles.active=prod
```

**方式四：环境变量**

```bash
export SPRING_PROFILES_ACTIVE=prod
```

**Nacos 多环境**：通过 Namespace 隔离（见第五章）。

### 2.4 配置加载完整顺序

```
启动
  │
  ├─ ① 读取 application.yml（按优先级顺序扫描 4 个目录）
  │
  ├─ ② 读取 application-{profile}.yml
  │
  ├─ ③ 读取环境变量
  │
  ├─ ④ 读取命令行参数
  │
  ├─ ⑤ 如果有 Nacos Config
  │     ├─ 从 Nacos 拉取共享配置（shared-configs / extension-configs）
  │     └─ 从 Nacos 拉取应用专属配置（${spring.application.name}.yaml）
  │     └─ Nacos 配置优先级 > 本地配置
  │
  └─ ⑥ 合并覆盖，生成最终 Environment
```

---

## 三、Nacos 服务注册与发现

### 3.1 为什么需要服务注册中心

微服务架构中，服务 A 要调用服务 B：

```
没有注册中心：
  服务 A 硬编码服务 B 的 IP:Port
  → 服务 B 换机器了？改代码
  → 服务 B 扩容了？改代码
  → 服务 B 挂了？A 不知道

有注册中心：
  服务 B 启动时向注册中心注册自己的地址
  服务 A 向注册中心查询服务 B 有哪些可用实例
  → 动态感知、自动剔除故障节点
```

### 3.2 Nacos 核心概念

```
┌─────────────────────────────────────┐
│              Nacos Server           │
│                                     │
│  Namespace（命名空间）               │
│    └─ Group（分组）                  │
│         └─ Service（服务）           │
│              └─ Cluster（集群）       │
│                   └─ Instance（实例） │
└─────────────────────────────────────┘
```

| 概念 | 说明 | 默认值 |
|------|------|--------|
| Namespace | 环境隔离，如 dev/test/prod | public |
| Group | 服务分组，如 DEFAULT_GROUP | DEFAULT_GROUP |
| Service | 服务名，如 user-service | - |
| Cluster | 集群，如同一机房内的实例 | DEFAULT |
| Instance | 具体的服务实例（IP + Port） | - |

### 3.3 服务注册流程

服务提供者启动时的注册流程：

```
服务启动
  │
  ▼
① NacosAutoServiceRegistration 监听 WebServerInitializedEvent
  │  （Web 容器初始化完成后触发）
  │
  ▼
② 构造 Registration 对象
  │  { serviceName: "user-service",
  │    ip: "192.168.1.100",
  │    port: 8080,
  │    weight: 1,
  │    healthy: true,
  │    ephemeral: true }
  │
  ▼
③ 调用 NamingService.registerInstance()
  │
  ▼
④ 发送 HTTP POST 请求到 Nacos Server
  │  POST /nacos/v1/ns/instance
  │  ?serviceName=user-service
  │  &ip=192.168.1.100
  │  &port=8080
  │
  ▼
⑤ Nacos Server 收到请求
  ├─ 校验参数
  ├─ 将实例加入内存中的服务注册表
  ├─ 如果是持久化实例，写入 Derby/MySQL
  ├─ 如果是临时实例（默认），只放内存
  └─ 返回成功
  │
  ▼
⑥ 客户端启动心跳定时任务
  │  每 5 秒发送一次心跳（PUT /nacos/v1/ns/instance/beat）
  │  告诉 Nacos Server "我还活着"
```

**心跳机制详解：**

```
客户端                          Nacos Server
  │                                │
  │  PUT /instance/beat (每5秒)    │
  │──────────────────────────────>│
  │                                │  更新实例最后心跳时间
  │                                │
  │  200 OK (返回下次间隔)         │
  │<──────────────────────────────│
  │                                │
  │  ... 持续发送心跳 ...          │
  │                                │
  │  【如果客户端宕机，心跳停止】    │
  │                                │
  │                                │  15秒未收到心跳 → 标记不健康
  │                                │  30秒未收到心跳 → 剔除实例
```

### 3.4 服务发现流程

服务消费者调用时的发现流程：

```
服务消费者启动
  │
  ▼
① @LoadBalanced RestTemplate 或 OpenFeign 拦截请求
  │
  ▼
② 调用 NamingService.selectInstances("user-service")
  │
  ▼
③ 检查本地缓存（内存）是否有该服务的实例列表
  │
  ├─ 有缓存 → 直接使用
  │
  └─ 无缓存 → 发送 HTTP GET 请求到 Nacos Server
       GET /nacos/v1/ns/instance/list?serviceName=user-service
       │
       ▼
     Nacos Server 返回健康实例列表
       │
       ▼
     存入本地缓存，并订阅该服务
       │
       ▼
     建立长连接，Nacos 推送实例变更

请求发送时：
  │
  ▼
④ 从缓存的实例列表中，按负载均衡策略选一个
  │  （默认轮询 RoundRobin）
  │
  ▼
⑤ 发起 HTTP 调用
```

**核心：服务发现 = 本地缓存 + 长连接推送 + 定时拉取兜底**

### 3.5 健康检查机制

Nacos 对临时实例和持久化实例的健康检查方式不同：

| 类型 | 健康检查方式 | 适用场景 |
|------|-------------|----------|
| 临时实例（默认） | 客户端心跳（5秒一次） | 大多数微服务 |
| 持久化实例 | Nacos Server 主动探测（TCP/HTTP） | 数据库、中间件等 |

```yaml
# 配置为持久化实例（心跳由服务端主动探测）
spring:
  cloud:
    nacos:
      discovery:
        ephemeral: false  # 默认是 true（临时实例）
```

**临时实例健康状态流转：**

```
健康 ──(15秒无心跳)──▶ 不健康 ──(30秒无心跳)──▶ 剔除
                                              （从实例列表中删除）
```

### 3.6 服务下线与临时实例

**正常下线：**

```java
// 服务关闭时，Spring 容器触发销毁回调
// NacosAutoServiceRegistration 监听到事件后：
namingService.deregisterInstance("user-service", ip, port);
```

```
服务关闭
  │
  ▼
发送 deregister 请求到 Nacos Server
  │
  ▼
Nacos Server 立即将实例从列表中移除
  │
  ▼
推送变更通知给所有订阅该服务的消费者
```

**异常下线（宕机、断网）：**

```
服务宕机
  │
  ▼
心跳停止
  │
  ▼
Nacos Server 15秒未收到心跳 → 标记不健康
  │
  ▼
30秒未收到心跳 → 自动剔除实例
  │
  ▼
推送变更给消费者 → 消费者更新本地缓存
```

**注意**：临时实例的下线时间 = 心跳间隔 × 超时倍数。默认 5秒 × 3 = 15秒标记不健康，30秒剔除。

---

## 四、Nacos 配置中心

### 4.1 配置中心解决什么问题

传统配置文件的问题：

| 问题 | 说明 |
|------|------|
| 修改配置需重启 | 每次改配置都要重新部署 |
| 多环境管理混乱 | dev/test/prod 各一份配置，容易搞混 |
| 配置分散 | 每个服务各自维护，没有统一视图 |
| 无法热更新 | 修改了配置但正在运行的服务拿不到 |

Nacos Config 解决了这些问题：统一存储、热更新、多环境隔离。

### 4.2 集成 Nacos Config

**引入依赖：**

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

**bootstrap.yml（注意：Nacos Config 配置必须放在 bootstrap.yml）：**

```yaml
spring:
  application:
    name: user-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml          # 配置文件格式
        namespace: dev-namespace-id   # 命名空间
        group: DEFAULT_GROUP          # 分组
```

**Nacos 上配置的 Data ID 规则：**

```
${prefix}-${spring.profile.active}.${file-extension}
```

例如：`user-service-dev.yaml`

如果不指定 profile，则是：`user-service.yaml`

### 4.3 动态刷新原理

Nacos Config 动态刷新的完整流程：

```
Nacos Server                          客户端应用
  │                                     │
  │  配置被修改（控制台编辑并发布）        │
  │                                     │
  │  推送变更通知（HTTP 长轮询）         │
  │<────────────────────────────────────│
  │                                     │
  │  客户端收到变更的 Data ID            │
  │                                     │
  │  拉取最新配置内容                    │
  │────────────────────────────────────>│
  │                                     │
  │  返回最新配置                        │
  │<────────────────────────────────────│
  │                                     │
  │  客户端更新 Environment             │
  │  发布 EnvironmentChangeEvent         │
  │  发布 RefreshEvent                   │
  │                                     │
  │  @RefreshScope 的 Bean 被销毁重建    │
  │  @ConfigurationProperties 自动刷新   │
```

**HTTP 长轮询机制**：

```
客户端发起长轮询请求：
  GET /nacos/v1/cs/configs/listening?configKeys=...
  超时时间 = 30秒

服务端处理：
  ├─ 配置有变更 → 立即返回变更的 Data ID
  └─ 配置无变更 → 挂起请求（hold 住），30秒后返回空
       （期间如果配置变了，立即返回）

客户端收到响应：
  ├─ 有变更 → 拉取最新配置，刷新 Bean
  └─ 无变更 → 立即发起下一轮长轮询
```

**为什么要用长轮询而不是 WebSocket？**
- 长轮询实现简单，不依赖额外协议
- 服务端挂起请求不会占用线程（Nacos 用异步 Servlet 实现）
- 兼容性好，防火墙不会拦截

### 4.4 @RefreshScope 机制

```java
@RestController
@RefreshScope  // 让这个 Bean 支持动态刷新
public class ConfigController {

    @Value("${my.config.value}")
    private String value;

    @GetMapping("/config")
    public String getConfig() {
        return value;  // Nacos 修改后，下次访问这里会拿到新值
    }
}
```

**@RefreshScope 做了什么：**

```
① 被 @RefreshScope 标注的 Bean 不会被直接放入 Spring 容器
② 而是被一个代理对象替代（ScopedProxy）
③ 代理对象内部持有一个缓存，第一次调用时从容器获取真实 Bean 并缓存
④ 配置变更时，@RefreshScope 清空缓存
⑤ 下次调用代理对象时，重新从容器获取新的 Bean（此时属性已是新值）
```

**@RefreshScope vs @ConfigurationProperties：**

| 特性 | @RefreshScope + @Value | @ConfigurationProperties |
|------|------------------------|--------------------------|
| 刷新方式 | 销毁重建 Bean | 直接更新字段 |
| 性能 | 较重（重建 Bean） | 轻量（只更新属性） |
| 推荐场景 | 少量配置项 | 大量配置项 |
| 是否需要注解 | 需要 | 不需要额外注解 |

**最佳实践**：推荐用 `@ConfigurationProperties`，不需要 @RefreshScope 也能自动刷新（Spring Cloud Alibaba 已内置支持）。

```java
@Component
@ConfigurationProperties(prefix = "app")
@Data
public class AppProperties {
    private String name;
    private int timeout;
    // Nacos 修改后，这些字段自动更新，不需要 @RefreshScope
}
```

### 4.5 配置监听底层实现

Nacos Config 在客户端的核心类是 `NacosContextRefresher`：

```java
// 简化版核心逻辑
public class NacosContextRefresher {

    public void registerNacosListener(String dataId, String group) {
        Listener listener = new Listener() {
            @Override
            public Executor getExecutor() {
                return null;  // 使用默认线程池
            }

            @Override
            public void receiveConfigInfo(String configInfo) {
                // 配置变更回调
                String content = configInfo;
                // 发布 Spring 事件
                applicationContext.publishEvent(new RefreshEvent(this));
            }
        };

        // 添加监听器
        ConfigService configService = NacosFactory.createConfigService(properties);
        configService.addListener(dataId, group, listener);
    }
}
```

`ConfigService.addListener()` 内部启动一个线程，通过长轮询检查配置变更，一旦变更就调用 `receiveConfigInfo()`。

---

## 五、命名空间与分组

### 5.1 命名空间 Namespace

Namespace 用于**环境隔离**，是最粗粒度的隔离级别。

```
Namespace: dev          ← 开发环境
  └─ user-service
  └─ order-service

Namespace: test         ← 测试环境
  └─ user-service
  └─ order-service

Namespace: prod         ← 生产环境
  └─ user-service
  └─ order-service
```

**使用方式：**

```yaml
spring:
  cloud:
    nacos:
      config:
        namespace: your-namespace-id    # 填 Namespace ID，不是名称
      discovery:
        namespace: your-namespace-id
```

**注意**：不指定 namespace 时默认是 `public` 命名空间。

### 5.2 分组 Group

Group 用于**同一环境下的服务/配置分组**，是 Namespace 之下的隔离。

```
Namespace: dev
  └─ Group: DEFAULT_GROUP
  │    └─ user-service
  │    └─ order-service
  │
  └─ Group: GROUP-A
  │    └─ user-service-v2
  │
  └─ Group: GROUP-B
       └─ order-service-v2
```

**使用场景**：
- 灰度发布：新版本服务放新 Group，旧版本放旧 Group
- 业务线隔离：不同业务线用不同 Group

```yaml
spring:
  cloud:
    nacos:
      config:
        group: MY_GROUP
      discovery:
        group: MY_GROUP
```

### 5.3 Data ID 规范

Data ID 是配置的唯一标识，格式为：

```
${prefix}-${spring.profile.active}.${file-extension}
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| prefix | spring.application.name | 可通过 `spring.cloud.nacos.config.prefix` 覆盖 |
| profile | 空 | 即激活的环境，如 dev |
| file-extension | properties | 可通过 `spring.cloud.nacos.config.file-extension` 配置 |

**示例：**

```yaml
# bootstrap.yml
spring:
  application:
    name: user-service
  profiles:
    active: dev
  cloud:
    nacos:
      config:
        file-extension: yaml
```

对应的 Data ID = `user-service-dev.yaml`

### 5.4 三者关系与隔离级别

```
最大隔离
  │
  ▼
Namespace（命名空间）    ← 环境级隔离（dev / test / prod）
  │
  ▼
Group（分组）           ← 业务级隔离（DEFAULT_GROUP / GROUP-A）
  │
  ▼
Data ID（配置ID）       ← 配置级隔离（user-service-dev.yaml）
  │
  ▼
最小隔离
```

**隔离优先级：Namespace > Group > Data ID**

**推荐实践：**

| 场景 | 使用方式 |
|------|----------|
| 区分开发/测试/生产 | 用 Namespace |
| 区分不同业务线 | 用 Group |
| 区分不同服务 | 用 Data ID（服务名） |
| 区分不同环境配置 | 用 Data ID 中的 profile |

---

## 六、生产实践

### 6.1 Nacos 集群部署

```
Nacos 集群（至少 3 节点）
  │
  ├─ 使用 MySQL 做持久化存储
  ├─ 使用 Nginx 做负载均衡
  └─ 使用 VIP 或域名访问

客户端配置：
  spring.cloud.nacos.config.server-addr=nacos-cluster.example.com:8848
  # 多个地址用逗号分隔
  # spring.cloud.nacos.config.server-addr=10.0.0.1:8848,10.0.0.2:8848,10.0.0.3:8848
```

### 6.2 配置加密

敏感配置（密码、密钥）不直接明文写在 Nacos 上：

```java
// 使用 Jasypt 加密
@SpringBootApplication
@EnableEncryptableProperties
public class Application { }

// application.yml
db:
  password: ENC(密文)  // Nacos 上存密文，客户端自动解密
```

### 6.3 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 配置不生效 | bootstrap.yml 没写对，或没引入 bootstrap 依赖 | 引入 `spring-cloud-starter-bootstrap` |
| 服务注册不上 | Nacos 地址不对，或 namespace 没配 | 检查 server-addr 和 namespace |
| 动态刷新不生效 | 没用 @RefreshScope 或 @ConfigurationProperties | 加对应注解 |
| 服务调用 404 | 服务名大小写不一致 | Nacos 服务名区分大小写 |
| 配置拉取慢 | 长轮询超时时间太长 | 调整 `config.long-poll.timeout` |
