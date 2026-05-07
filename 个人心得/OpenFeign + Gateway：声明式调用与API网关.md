# OpenFeign + Gateway：声明式调用与 API 网关

## 一、OpenFeign 声明式服务调用

### 1.1 为什么需要 Feign

在微服务架构中，服务 A 调用服务 B 通常需要用 `RestTemplate` 或 `HttpClient` 手写 HTTP 请求：

```java
// 传统方式：手动拼接 URL、序列化、处理响应
String url = "http://user-service/api/users/" + userId;
ResponseEntity<User> response = restTemplate.getForEntity(url, User.class);
```

**痛点**：
- URL 硬编码，服务名/路径散落各处
- 参数拼接容易出错
- 每个调用都要处理序列化、异常、重试
- 接口变更时调用方难以感知

Feign 的核心思想：**把 HTTP 调用抽象成接口方法调用**，像调用本地 Service 一样调用远程服务。

```java
@FeignClient(name = "user-service", path = "/api/users")
public interface UserClient {
    @GetMapping("/{id}")
    Result<User> getUserById(@PathVariable("id") Long id);
    
    @PostMapping
    Result<Long> createUser(@RequestBody UserDTO dto);
}

// 使用时直接注入
@Autowired
private UserClient userClient;

User user = userClient.getUserById(1L).getData();
```

### 1.2 @FeignClient 注解全解析

```java
@FeignClient(
    name = "user-service",           // 服务名（从 Nacos 获取地址）
    path = "/api/users",             // 统一前缀
    url = "${user.service.url:}",   // 直连 URL（调试用，覆盖服务发现）
    configuration = FeignConfig.class, // 自定义配置类
    fallback = UserClientFallback.class, // 降级实现
    fallbackFactory = UserClientFallbackFactory.class // 带异常信息的降级
)
```

**name vs serviceId vs value**：
- `name` 和 `value` 是同一个属性（`value` 是别名）
- `serviceId` 已废弃，等价于 `name`
- 配合 Nacos 使用时，`name` 必须是 Nacos 中注册的服务名

**contextId 的作用**：
```java
// 问题：两个 FeignClient 同路径同名接口会冲突
@FeignClient(name = "user-service", path = "/api/users")
public interface UserClientV1 { ... }

@FeignClient(name = "user-service", path = "/api/users")
public interface UserClientV2 { ... }
// 报错：Bean name 冲突

// 解决：用 contextId 区分
@FeignClient(name = "user-service", contextId = "userClientV1", path = "/api/users")
@FeignClient(name = "user-service", contextId = "userClientV2", path = "/api/users")
```

### 1.3 Feign 底层执行流程

```
方法调用 -> SynchronousMethodHandler -> 
  1. 构建 RequestTemplate（拼 URL、填 Header、序列化 Body）
  2. 通过 LoadBalancerClient 从 Nacos 获取实例列表并负载均衡
  3. 通过 Client.Default（底层 HttpURLConnection）或 OkHttp/Apache HttpClient 发送请求
  4. 解析 Response（反序列化为返回值）
  5. 如果失败 -> 走 DecoderError（默认抛 FeignException）
  6. 如果配置了 fallback -> 走降级逻辑
```

**核心组件**：

| 组件 | 作用 | 默认实现 |
|------|------|----------|
| Encoder | 请求体序列化 | `SpringEncoder` |
| Decoder | 响应体反序列化 | `SpringDecoder` |
| Contract | 解析 Feign 接口注解 | `SpringMvcContract` |
| Logger | 日志 | `Slf4jLogger` |
| Client | HTTP 客户端 | `Client.Default` |
| Retryer | 重试策略 | `Retryer.NEVER_RETRY` |
| ErrorDecoder | 异常解码 | `ErrorDecoder.Default` |

### 1.4 超时配置

Feign 的超时分两层：**Feign 层** 和 **底层 HTTP 客户端层**。两者取小。

```yaml
# Feign 层超时（推荐统一配置）
feign:
  client:
    config:
      default:              # default 对所有服务生效，也可指定服务名
        connectTimeout: 5000   # 连接超时 5s
        readTimeout: 10000     # 读取超时 10s
```

```java
// 也可以通过 Bean 配置
@Configuration
public class FeignConfig {
    @Bean
    public Request.Options options() {
        // 连接超时 5s，读取超时 10s
        return new Request.Options(5000, 10000);
    }
}
```

**超时与 Sentinel 的关系**：
- Feign 超时抛出 `FeignException`
- 如果同时配置了 Sentinel 熔断，超时会被 Sentinel 捕获并触发降级
- 建议：Feign 超时 > Sentinel 熔断超时，让 Sentinel 先触发降级

### 1.5 重试机制

Feign 默认不开启重试（`Retryer.NEVER_RETRY`）。

```java
// 方式一：简单重试
@Bean
public Retryer feignRetryer() {
    // 初始间隔 100ms，最大间隔 1s，最大重试次数 5
    return new Retryer.Default(100, 1000, 5);
}

// 方式二：自定义重试策略
public class CustomRetryer implements Retryer {
    private int attempt = 0;
    private static final int MAX_ATTEMPTS = 3;
    
    @Override
    public void continueOrPropagate(RetryableException e) {
        if (++attempt > MAX_ATTEMPTS) {
            throw e; // 超过最大次数，抛出
        }
        // 根据状态码决定是否重试
        if (e.status() == 503) {
            try {
                Thread.sleep(500L * attempt); // 递增等待
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        } else {
            throw e; // 非 503 不重试
        }
    }
}
```

**生产建议**：
- **Feign 层慎用重试**：重试 + 重试可能导致雪崩（调用方重试 * 调用方重试 = 请求量指数增长）
- 更好的做法：在 Sentinel 层配置慢调用比例降级 + 重试，或用 Spring Retry 在业务层控制
- 只有**幂等接口**（GET/DELETE）才适合重试，POST/PUT 重试需要保证幂等性

### 1.6 拦截器（RequestInterceptor）

Feign 拦截器用于在请求发出前统一处理，典型场景：传递 Token、添加 TraceId、统一 Header。

```java
@Component
public class FeignTokenInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 从当前请求上下文获取 Token，透传到下游服务
        ServletRequestAttributes attrs = 
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attrs != null) {
            HttpServletRequest request = attrs.getRequest();
            String token = request.getHeader("Authorization");
            if (StringUtils.hasText(token)) {
                template.header("Authorization", token);
            }
        }
        
        // 添加链路追踪 ID
        template.header("X-Trace-Id", MDC.get("traceId"));
    }
}
```

**常见坑——异步线程中 Feign 拦截器拿不到 RequestContextHolder**：

```java
// 问题场景：在 @Async 方法中调用 Feign
@Async
public void asyncCall() {
    userClient.getUserById(1L); // RequestContextHolder 为 null
}

// 解决：在主线程传递 RequestAttributes
@Async
public void asyncCall() {
    RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
    try {
        RequestContextHolder.setRequestAttributes(attrs);
        userClient.getUserById(1L);
    } finally {
        RequestContextHolder.resetRequestAttributes();
    }
}
```

### 1.7 日志级别

```yaml
feign:
  client:
    config:
      user-service:
        loggerLevel: FULL    # 日志级别
  logging:
    level:
      com.example.client.UserClient: DEBUG  # 必须开启 DEBUG
```

| 级别 | 内容 | 适用场景 |
|------|------|----------|
| NONE | 不记录日志 | 生产环境（默认） |
| BASIC | 记录请求方法、URL、响应状态码、耗时 | 生产环境推荐 |
| HEADERS | BASIC + 请求/响应头 | 调试 Header 问题 |
| FULL | HEADERS + 请求体 + 响应体 | 开发环境调试 |

### 1.8 降级与容错（fallback）

```java
// 方式一：简单 fallback（拿不到异常信息）
@Component
public class UserClientFallback implements UserClient {
    @Override
    public Result<User> getUserById(Long id) {
        return Result.fail("user-service 服务不可用，请稍后重试");
    }
}

// 方式二：fallbackFactory（可以拿到异常，适合打日志/告警）
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {
    @Override
    public UserClient create(Throwable cause) {
        log.error("user-service 调用失败: {}", cause.getMessage(), cause);
        return new UserClient() {
            @Override
            public Result<User> getUserById(Long id) {
                return Result.fail("user-service 降级: " + cause.getMessage());
            }
        };
    }
}
```

**触发降级的条件**：
- 网络异常（连接超时、DNS 解析失败）
- 返回非 2xx 状态码
- Sentinel 熔断（配合 Sentinel 时）

## 二、Spring Cloud Gateway 网关

### 2.1 网关的定位

```
客户端 -> [Gateway 网关] -> [微服务 A / 微服务 B / 微服务 C]
```

网关的核心职责：
1. **统一入口**：客户端只和网关交互，不直接访问微服务
2. **路由转发**：根据路径/Header 将请求路由到对应服务
3. **横切关注点**：鉴权、限流、CORS、日志、灰度发布
4. **协议转换**：HTTP -> gRPC / WebSocket 等

### 2.2 Gateway vs Zuul

| 维度 | Zuul 1.x | Spring Cloud Gateway |
|------|----------|---------------------|
| 模型 | 阻塞 I/O（同步） | 响应式（WebFlux + Netty，异步非阻塞） |
| 性能 | 线程池模型，吞吐有限 | 事件循环模型，高吞吐低延迟 |
| 开发语言 | Java | Java（Reactor） |
| 维护状态 | 基本停止更新 | Spring 官方主推 |

### 2.3 核心概念：路由、谓词、过滤器

```
Route（路由）= Predicate（谓词/匹配条件）+ Filter（过滤器）+ URI（目标地址）
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_route                    # 路由 ID（唯一）
          uri: lb://user-service            # 目标地址（lb=负载均衡）
          predicates:
            - Path=/api/users/**            # 谓词：路径匹配
            - Method=GET,POST               # 谓词：方法匹配
          filters:
            - StripPrefix=1                 # 过滤器：去掉前缀 /api
            - name: RequestRateLimiter      # 过滤器：限流
              args:
                redis-rate-limiter.replenishRate: 10
```

**执行流程**：

```
请求 -> HandlerMapping（匹配路由）-> 
  -> FilteringWebHandler（执行过滤器链）->
    -> Global Filters（全局过滤器）
    -> Route Filters（路由过滤器）
    -> 转发到目标服务
    -> 响应经过反向过滤器链返回
```

### 2.4 内置谓词（Predicate）大全

| 谓词 | 示例 | 说明 |
|------|------|------|
| Path | `Path=/api/users/**` | 路径匹配 |
| Method | `Method=GET,POST` | HTTP 方法 |
| Header | `Header=X-Request-Id, \d+` | Header 存在且匹配正则 |
| Query | `Query=token, abc.+` | 查询参数存在且匹配正则 |
| RemoteAddr | `RemoteAddr=192.168.1.0/24` | 来源 IP 段 |
| Host | `Host=**.example.com` | Host 头匹配 |
| Before | `Before=2026-01-01T00:00:00+08:00` | 时间之前 |
| After | `After=2026-01-01T00:00:00+08:00` | 时间之后 |
| Between | `Between=2026-01-01T00:00:00+08:00, 2026-12-31T23:59:59+08:00` | 时间区间 |
| Cookie | `Cookie=chocolate, ch.p` | Cookie 存在且匹配正则 |
| Weight | `Weight=group1, 80` | 权重路由（灰度发布用） |

**组合使用**（AND 关系）：
```yaml
predicates:
  - Path=/api/users/**
  - Method=GET
  - Header=X-User-Token, .+    # 必须有 Token Header
```

### 2.5 内置过滤器

| 过滤器 | 作用 | 示例 |
|--------|------|------|
| StripPrefix | 去掉路径前缀 | `StripPrefix=1`：/api/users -> /users |
| PrefixPath | 添加路径前缀 | `PrefixPath=/api`：/users -> /api/users |
| AddRequestHeader | 添加请求头 | `AddRequestHeader=X-Name, Value` |
| AddRequestParameter | 添加查询参数 | `AddRequestParameter=token, xxx` |
| AddResponseHeader | 添加响应头 | `AddResponseHeader=X-Request-Id, xxx` |
| SetPath | 替换路径 | `SetPath=/api/{segment}` |
| RedirectTo | 重定向 | `RedirectTo=302, https://example.com` |
| RequestSize | 请求体大小限制 | `RequestSize=5MB` |
| Retry | 重试 | 按状态码/次数重试 |

**自定义全局过滤器**：
```java
@Component
@Order(-1)  // 数字越小越先执行
public class AuthGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().toString();
        
        // 白名单直接放行
        if (path.startsWith("/api/public/")) {
            return chain.filter(exchange);
        }
        
        // 校验 Token
        String token = request.getHeaders().getFirst("Authorization");
        if (!StringUtils.hasText(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // 将用户信息传递给下游
        ServerHttpRequest modified = request.mutate()
            .header("X-User-Id", parseUserId(token))
            .build();
        
        return chain.filter(exchange.mutate().request(modified).build());
    }
}
```

### 2.6 CORS 跨域配置

Gateway 处理 CORS 有两种方式：

**方式一：配置文件**
```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
```

**方式二：自定义 Bean**（更灵活）
```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOriginPattern("*");
        config.addAllowedMethod("*");
        config.addAllowedHeader("*");
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = 
            new UrlBasedCorsConfigurationSource(new PathPatternParser());
        source.registerCorsConfiguration("/**", config);
        return new CorsWebFilter(source);
    }
}
```

**避坑**：Gateway 是 WebFlux 项目，不能用 Spring MVC 的 `@CrossOrigin` 或 `WebMvcConfigurer`。

### 2.7 网关统一鉴权

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    
    private static final Set<String> WHITE_LIST = Set.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/public/"
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().toString();
        
        // 1. 白名单放行
        if (WHITE_LIST.contains(path) || path.startsWith("/api/public/")) {
            return chain.filter(exchange);
        }
        
        // 2. 从 Header 获取 Token
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            return unauthorized(exchange, "未登录或Token无效");
        }
        
        // 3. 验证 Token（同步或调用认证服务）
        try {
            String userId = JwtUtil.parseToken(token.substring(7));
            // 将用户信息传递给下游
            ServerHttpRequest request = exchange.getRequest().mutate()
                .header("X-User-Id", userId)
                .header("X-User-Role", JwtUtil.getRole(token.substring(7)))
                .build();
            return chain.filter(exchange.mutate().request(request).build());
        } catch (Exception e) {
            return unauthorized(exchange, "Token验证失败: " + e.getMessage());
        }
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange, String msg) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
        DataBuffer buffer = exchange.getResponse()
            .bufferFactory().wrap(("{\"code\":401,\"msg\":\"" + msg + "\"}").getBytes());
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
    
    @Override
    public int getOrder() {
        return -100;  // 优先级高于普通路由过滤器
    }
}
```

### 2.8 网关限流（基于 Redis）

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10    # 每秒补充令牌数
                redis-rate-limiter.burstCapacity: 20    # 令牌桶容量
                redis-rate-limiter.requestedTokens: 1   # 每次请求消耗令牌数
                key-resolver: "#{@ipKeyResolver}"       # 限流维度
```

```java
// 按 IP 限流
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
}

// 按用户限流
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    );
}

// 按路径限流
@Bean
public KeyResolver pathKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getPath().toString());
}
```

### 2.9 动态路由（Nacos 配置中心）

```yaml
# 配合 Nacos 实现动态路由（不用重启网关）
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true  # 自动根据 Nacos 服务生成路由
      routes:
        # 也可以从 Nacos 配置中心动态拉取路由配置
```

```java
// 自定义动态路由加载器：监听 Nacos 配置变化
@Component
public class DynamicRouteService implements ApplicationEventPublisherAware {
    
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    
    private ApplicationEventPublisher publisher;
    
    /**
     * 当 Nacos 配置变更时，刷新路由
     */
    public void refreshRoutes(List<RouteDefinition> routes) {
        // 1. 删除旧路由
        routeDefinitionWriter.getRouteDefinitions()
            .map(RouteDefinition::getId)
            .flatMap(routeDefinitionWriter::delete)
            .subscribe();
        
        // 2. 添加新路由
        routes.forEach(route -> 
            routeDefinitionWriter.save(Mono.just(route)).subscribe()
        );
        
        // 3. 发布路由刷新事件
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
    
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }
}
```

## 三、OpenFeign + Gateway 协同架构

### 3.1 典型请求链路

```
客户端 -> [Gateway] -> 统一鉴权 -> 限流 -> 路由转发 -> [user-service]
                                                    -> 内部调用 -> [order-service]
                                                              -> Feign -> [product-service]
```

### 3.2 链路追踪传递

```java
// Gateway 层：生成 TraceId 并传递
@Component
public class TraceGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String traceId = exchange.getRequest().getHeaders().getFirst("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }
        
        ServerHttpRequest request = exchange.getRequest().mutate()
            .header("X-Trace-Id", traceId)
            .build();
        
        // MDC 绑定，日志输出带 TraceId
        MDC.put("traceId", traceId);
        
        return chain.filter(exchange.mutate().request(request).build())
            .doFinally(signal -> MDC.remove("traceId"));
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;  // 最优先执行
    }
}
```

### 3.3 Gateway 中调用 Feign 的问题

**问题**：Gateway 基于 WebFlux（响应式），Feign 基于 Spring MVC（阻塞式），直接在 Gateway 中用 Feign 会阻塞事件循环。

**解决方案**：
1. **推荐**：在独立的认证服务中做鉴权，Gateway 通过 HTTP 调用认证服务
2. Gateway 中用 `WebClient`（响应式客户端）替代 Feign
3. Gateway 只做路由+鉴权，Feign 放在业务服务间使用

## 四、生产实践与避坑

### 4.1 Feign 调用超时排查

```
现象：Feign 调用偶尔超时
排查思路：
1. 看是哪个服务超时（日志中找 service name）
2. 查 Nacos 中该服务的实例健康状态
3. 查目标服务的 GC 日志（Full GC 导致 STW）
4. 查数据库慢查询（下游服务查 DB 慢）
5. 查网络延迟（同可用区 vs 跨可用区）
```

### 4.2 Feign 传递文件的坑

Feign 默认不支持 `multipart/form-data`（文件上传）。需要额外依赖：

```xml
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
</dependency>
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form-spring</artifactId>
</dependency>
```

```java
@Configuration
public class FeignMultipartConfig {
    @Bean
    @Primary
    @Scope("prototype")
    public Encoder feignEncoder() {
        return new SpringFormEncoder(new SpringEncoder(ObjectFactory::new, ...));
    }
}
```

### 4.3 Gateway 路由匹配优先级

```
当多个路由都能匹配同一个请求时，按 routes 列表中的顺序，第一个匹配的路由生效。
```

```yaml
routes:
  - id: route_a
    uri: lb://service-a
    predicates:
      - Path=/api/**        # 会先匹配到
    
  - id: route_b
    uri: lb://service-b
    predicates:
      - Path=/api/users/**  # 更精确，但排在后面，不会被匹配到
```

**正确做法**：更精确的路由放在前面。

### 4.4 Gateway 响应式编程注意

- Gateway 内部是 Reactor 响应式模型，不要用阻塞 API（如 `Thread.sleep()`、同步 JDBC）
- 自定义过滤器必须返回 `Mono<Void>` 或 `Flux<Void>`
- 如果需要调用阻塞型服务（如同步查 DB），用 `publishOn(Schedulers.boundedElastic())` 切换线程
