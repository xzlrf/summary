# JVM & 排查：内存模型、GC、Arthas 与线上问题排查

## 一、JVM 内存结构

### 1.1 运行时数据区总览

```
JVM 内存 = 线程私有 + 线程共享

线程私有：
  1. 程序计数器（Program Counter Register）
  2. 虚拟机栈（Java Virtual Machine Stack）
  3. 本地方法栈（Native Method Stack）

线程共享：
  1. 堆（Heap）
  2. 方法区（Method Area / 元空间 Metaspace）
```

### 1.2 堆（Heap）

**最大的一块内存**，存放对象实例，是 GC 的主要区域。

```
堆内存分代结构（JDK 8+）：

Heap
├── 新生代（Young Generation）—— 1/3 堆空间
│   ├── Eden 区（8/10）—— 新对象出生地
│   └── Survivor 区（2/10）—— S0（From）和 S1（To）
│
└── 老年代（Old Generation）—— 2/3 堆空间
    └── 经过多次 GC 仍然存活的对象
```

**对象分配与晋升过程**：

```
1. 新对象在 Eden 区分配
2. Eden 满了 -> Minor GC -> 存活对象移到一个 Survivor 区
3. 下次 Minor GC -> Eden + 上一个 Survivor 存活 -> 移到另一个 Survivor
4. 每次 Minor GC 存活对象的年龄 +1
5. 年龄达到阈值（默认 15）-> 晋升到老年代
6. 大对象（如大数组）直接进入老年代
```

```java
// 查看堆内存配置
// -Xms256m    初始堆大小
// -Xmx512m    最大堆大小（生产建议 -Xms = -Xmx）
// -Xmn128m    新生代大小
// -XX:SurvivorRatio=8  Eden/Survivor 比例
// -XX:MaxTenuringThreshold=15  晋升老年代年龄阈值
```

### 1.3 虚拟机栈（Java Stack）

**每个线程一个栈**，存放方法执行的栈帧（局部变量表、操作数栈、动态链接、返回地址）。

```
调用方法 -> 入栈（压入一个栈帧）
方法返回 -> 出栈（弹出栈帧）

栈帧结构：
  局部变量表：方法参数、局部变量（this 也算）
  操作数栈：执行字节码的临时存储
  动态链接：方法符号引用
  返回地址：方法执行完后回到哪里
```

**常见问题**：

```
StackOverflowError：
  原因：递归太深或方法调用层级过多
  解决：检查递归逻辑，增大 -Xss（线程栈大小，默认 1M）
```

### 1.4 元空间（Metaspace）

**JDK 8 替换了永久代（PermGen）**，存放类元信息、常量池、方法字节码等。

```
永久代 vs 元空间：
  永久代：在堆内，大小固定（-XX:MaxPermSize），容易 OOM
  元空间：在本地内存（不在堆），默认无上限，受系统内存限制

配置：
  -XX:MetaspaceSize=256m    初始大小
  -XX:MaxMetaspaceSize=512m  最大大小（建议设置，防止无限增长）
```

**元空间 OOM 场景**：
- 大量动态代理类生成（CGLIB、Groovy 脚本）
- 频繁加载类但不卸载（类加载器泄漏）
- 应用频繁热部署/热重启

### 1.5 直接内存（Direct Memory）

**不在 JVM 规范中**，但实际存在。NIO 的 `ByteBuffer.allocateDirect()` 分配的内存。

```java
// 直接内存（堆外内存）
ByteBuffer buffer = ByteBuffer.allocateDirect(10 * 1024 * 1024);  // 10MB
// 不受 GC 管理，需要手动释放（或依赖 GC 间接回收）

// 配置限制
// -XX:MaxDirectMemorySize=512m  最大直接内存
```

### 1.6 各区域 OOM 总结

| 区域 | 异常 | 常见原因 | 排查命令 |
|------|------|----------|----------|
| 堆 | `OutOfMemoryError: Java heap space` | 内存泄漏、大对象、并发创建过多 | `jmap -heap` |
| 栈 | `StackOverflowError` | 递归太深 | 看线程 dump |
| 元空间 | `OutOfMemoryError: Metaspace` | 类加载器泄漏、动态代理过多 | `jmap -clstats` |
| 直接内存 | `OutOfMemoryError: Direct buffer memory` | NIO 直接缓冲区未释放 | `-XX:NativeMemoryTracking` |

## 二、垃圾回收（GC）

### 2.1 判断对象是否存活

**可达性分析算法**：

```
从 GC Roots 开始向下搜索，能被 GC Roots 到达的对象是活的，否则是垃圾。

GC Roots 包括：
  1. 虚拟机栈中局部变量引用的对象
  2. 方法区中静态属性引用的对象
  3. 方法区中常量引用的对象
  4. 本地方法栈中 JNI 引用的对象
```

**引用类型**：

| 类型 | 回收时机 | 示例 |
|------|----------|------|
| 强引用 | 永远不会被回收 | `Object obj = new Object()` |
| 软引用 | 内存不足时才回收 | `SoftReference`（缓存用） |
| 弱引用 | 下次 GC 就回收 | `WeakReference`（ThreadLocal） |
| 虚引用 | 随时可能被回收 | `PhantomReference`（监控用） |

### 2.2 GC 算法

| 算法 | 原理 | 优点 | 缺点 | 使用区域 |
|------|------|------|------|----------|
| 标记-清除 | 标记存活对象，清除未标记的 | 简单 | 内存碎片 | 老年代 |
| 标记-复制 | 标记后复制到另一半空间 | 无碎片 | 浪费一半空间 | 新生代 |
| 标记-整理 | 标记后整理，把存活对象移到一端 | 无碎片 | 移动开销 | 老年代 |

### 2.3 垃圾收集器

```
新生代收集器：
  Serial：单线程，适合客户端应用
  ParNew：Serial 的多线程版，配合 CMS 使用
  Parallel Scavenge：多线程，注重吞吐量（JDK 8 默认）
  G1：分区收集，可预测停顿（JDK 9+ 默认）

老年代收集器：
  CMS：低延迟，但有浮动垃圾和碎片（已废弃）
  Parallel Old：Parallel 的老年代版
  G1：全区域收集
  ZGC：超低延迟（JDK 15+）
```

### 2.4 G1 收集器（Garbage First）

**核心思想**：把堆划分为多个大小相等的 Region，按回收价值排序，优先回收垃圾最多的 Region。

```
G1 的 Region 类型：
  [E] Eden
  [S] Survivor
  [O] Old
  [H] Humongous（超大对象区域，>= 半个 Region）

G1 的两种停顿：
  Young GC：只回收 Eden 区
  Mixed GC：回收 Eden + 部分 Old Region（由 GC 策略决定）
```

```
G1 回收过程：
  1. 初始标记（Stop-The-World）：标记 GC Roots 直接关联的对象
  2. 并发标记：与用户线程同时运行，遍历对象图
  3. 最终标记（Stop-The-World）：修正并发标记期间的变动
  4. 筛选回收（Stop-The-World）：选择回收价值最高的 Region
```

```java
// 启用 G1
// -XX:+UseG1GC
// -XX:MaxGCPauseMillis=200  最大停顿时间目标
// -XX:G1HeapRegionSize=16m  Region 大小
```

### 2.5 Full GC 触发条件

| 条件 | 说明 |
|------|------|
| 老年代空间不足 | 大对象直接进入老年代，或晋升过多 |
| 元空间不足 | 类元信息满了 |
| Minor GC 后晋升失败 | Survivor 放不下，老年代也放不下 |
| 显式调用 `System.gc()` | 建议 JVM 执行 Full GC（默认启用） |

### 2.6 GC 日志解读

```
[GC (Allocation Failure) [PSYoungGen: 51200K->8384K(60928K)] 
       51200K->8400K(200704K), 0.0153941 secs]

解读：
  PSYoungGen: 51200K->8384K(60928K)
    新生代 GC 前 51200K -> GC 后 8384K，新生代总容量 60928K
  51200K->8400K(200704K)
    堆 GC 前 51200K -> GC 后 8400K，堆总容量 200704K
  0.0153941 secs：GC 耗时 15ms
```

## 三、Arthas 使用

### 3.1 什么是 Arthas

阿里开源的 Java 诊断工具，**无需重启应用**即可在线排查。

```
适用场景：
  1. 线上代码有问题，但本地无法复现
  2. 想知道某个方法被谁调用了
  3. 想看方法入参和返回值
  4. CPU 飙高，想知道哪个线程在忙
  5. 想看某个对象的字段值
```

### 3.2 安装与启动

```bash
# 下载
curl -O https://arthas.aliyun.com/arthas-boot.jar

# 启动（选择要诊断的 Java 进程）
java -jar arthas-boot.jar

# 或者直接连接
java -jar arthas-boot.jar <pid>
```

### 3.3 常用命令

#### dashboard（总览）

```bash
dashboard
# 显示：
#   - 线程总数、CPU 使用
#   - 内存使用情况（堆/非堆）
#   - GC 统计
#   - 最忙的线程
```

#### thread（线程分析）

```bash
# 查看最忙的 N 个线程
thread -n 3

# 查看指定线程的堆栈
thread <thread-id>

# 查找死锁
thread -b

# 查看阻塞最多的线程
thread -i 1000
```

#### jad（反编译）

```bash
# 反编译某个类（确认线上跑的代码是不是最新的）
jad com.example.service.OrderService

# 反编译指定方法
jad com.example.service.OrderService createOrder
```

#### watch（方法观测）

```bash
# 观察方法入参和返回值
watch com.example.service.OrderService createOrder '{params,returnObj}' -x 2

# 观察异常
watch com.example.service.OrderService createOrder '{params,throwExp}' -e

# 观察执行时间
watch com.example.service.OrderService createOrder '{params,returnObj}' '#cost>100'
```

```
参数说明：
  -x 2：展开深度 2 层
  -e：只看异常
  '#cost>100'：只看耗时超过 100ms 的调用
```

#### trace（方法调用链追踪）

```bash
# 追踪方法内部调用链路和耗时
trace com.example.service.OrderService createOrder

# 只看耗时超过 100ms 的分支
trace com.example.service.OrderService createOrder '#cost>100'
```

```
输出示例：
`---[OrderService.createOrder]
    +---[0.05ms] OrderMapper.insert()
    +---[45.23ms] StockClient.deduct()
    |   `---[43.10ms] RestTemplate.postForObject()  <-- 这里慢
    +---[0.02ms] OrderClient.updateStatus()
```

#### monitor（方法监控）

```bash
# 统计方法的调用频率、成功/失败次数
monitor -c 5 com.example.service.OrderService createOrder
# 每 5 秒输出一次
```

#### getstatic / ognl（查看静态变量 / 执行表达式）

```bash
# 查看静态变量
getstatic com.example.config.GlobalConfig MAX_RETRY

# 查看 Spring Bean
ognl '@org.springframework.web.context.support.WebApplicationContextUtils@getWebApplicationContext(@javax.servlet.http.HttpServletRequest@).getBean("orderService")'
```

#### heapdump（堆转储）

```bash
# dump 堆内存到文件
heapdump /tmp/heapdump.hprof

# 只 dump 存活对象
heapdump --live /tmp/heapdump.hprof
```

## 四、线上问题排查

### 4.1 CPU 100% 排查思路

```
步骤：

1. top 命令找到 CPU 最高的进程 PID
   top

2. 找到该进程下 CPU 最高的线程 TID
   top -Hp <PID>

3. 把 TID 转换为十六进制
   printf "%x\n" <TID>
   # 得到 nid（如 0x1a2b）

4. jstack 导出线程堆栈，找到对应线程
   jstack <PID> | grep -A 20 "nid=0x1a2b"

5. 分析堆栈：看是哪个方法在疯狂执行

6. 如果是 Arthas 环境：
   thread -n 3  # 直接看最忙的 3 个线程
   trace 类名 方法名  # 追踪耗时
```

```
常见 CPU 高的原因：
  1. 死循环（while(true) 没有退出条件）
  2. 频繁 GC（Full GC 导致 CPU 飙高，用 GC 日志确认）
  3. 正则表达式回溯（如 `(a+)+b` 匹配长字符串）
  4. 大量计算（加密、序列化）
  5. 线程过多（上下文切换开销）
```

### 4.2 OOM 排查思路

#### 4.2.1 堆内存 OOM

```
1. 确认 OOM 类型
   java.lang.OutOfMemoryError: Java heap space

2. 分析 Heap Dump
   jmap -dump:live,format=b,file=/tmp/heap.hprof <PID>
   或使用 MAT（Memory Analyzer Tool）打开 .hprof 文件

3. MAT 分析步骤：
   - Histogram：按类统计对象数量和内存占比
   - Dominator Tree：找到最大的对象支配树
   - GC Roots：找到泄漏对象的引用链

4. 常见泄漏场景：
   - 静态集合持续添加对象（static Map/List 无限增长）
   - 连接池未关闭（DB 连接、HTTP 连接）
   - ThreadLocal 未 remove（线程复用导致数据残留）
   - 事件监听器/回调函数未注销
   - 缓存无过期时间（Caffeine/Guava Cache 未设大小上限）
```

```java
// ThreadLocal 泄漏示例
public class ThreadLocalLeak {
    private static final ThreadLocal<UserContext> CONTEXT = new ThreadLocal<>();
    
    public void handleRequest(UserContext ctx) {
        CONTEXT.set(ctx);
        try {
            processRequest();
        } finally {
            CONTEXT.remove();  // 必须 remove！否则线程池复用时会泄漏
        }
    }
}
```

#### 4.2.2 元空间 OOM

```
1. 确认 OOM 类型
   java.lang.OutOfMemoryError: Metaspace

2. 排查思路：
   jmap -clstats <PID>  # 查看类加载器统计
   jmap -permstat <PID>  # 查看永久代/元空间对象

3. 常见原因：
   - 动态代理类无限生成（CGLIB 每次生成新的类）
   - Groovy/JS 脚本引擎频繁加载类
   - OSGi 热部署导致类加载器泄漏

4. 解决：
   - 限制 -XX:MaxMetaspaceSize
   - 缓存动态代理类（CGLIB 的 Key 要正确实现 equals/hashCode）
   - 脚本引擎复用（不每次都创建新引擎）
```

#### 4.2.3 直接内存 OOM

```
1. 确认 OOM 类型
   java.lang.OutOfMemoryError: Direct buffer memory

2. 排查：
   -XX:NativeMemoryTracking=detail  # 开启本地内存追踪
   jcmd <PID> VM.native_memory summary  # 查看本地内存分布

3. 常见原因：
   - NIO ByteBuffer.allocateDirect() 未释放
   - Netty 的 DirectByteBuf 未 release
   - JDBC 驱动使用堆外内存

4. 解决：
   - Netty 中确保 ReferenceCounted.release() 被调用
   - 使用 try-with-resources 自动关闭 NIO 资源
```

### 4.3 线上问题排查万能模板

```
1. 先看日志
   - 有没有异常堆栈
   - 有没有 Full GC 日志
   - 有没有慢 SQL 日志

2. 看监控
   - CPU、内存、磁盘、网络
   - JVM 指标（GC 频率、线程数、堆使用率）
   - 应用指标（QPS、RT、错误率）

3. 定位方向
   CPU 高 -> thread -n 3 -> trace 慢方法
   内存高 -> jmap dump -> MAT 分析
   响应慢 -> Arthas trace/watch -> 找瓶颈
   死锁 -> jstack / thread -b

4. 临时止血
   - 扩容/重启（最快但不治本）
   - 限流/降级（保护系统）
   - 回滚（如果是新上线导致的）

5. 根因分析
   - 代码 Review
   - MAT 分析
   - 复现问题
   - 修复 + 测试 + 上线
```
