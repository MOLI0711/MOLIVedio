# CAT Transaction & Metrics 完整使用指南
## 📚 目录
1. [核心概念](#核心概念)
2. [CAT Transaction 使用指南](#cat-transaction-使用指南)
3. [CAT Event 使用指南](#cat-event-使用指南)
4. [Metrics 使用指南](#metrics-使用指南)
5. [组合使用模式](#组合使用模式)
6. [项目最佳实践](#项目最佳实践)
7. [监控设计原则](#监控设计原则)
8. [常见问题](#常见问题)
---
## 核心概念
### 监控工具对比
| 工具 | 主要用途 | 核心指标 | 使用场景 |
|------|---------|---------|---------|
| **CAT Transaction** | 监控一段代码的运行情况 | 次数、QPS、错误率、响应时间（Avg/TP90/TP99） | 核心业务逻辑、RPC调用、DB操作 |
| **CAT Event** | 记录事件发生次数 | 次数、错误率、QPS | 业务事件、条件分支、状态变更 |
| **Metrics Counter** | 累计计数 | 累计次数 | 请求次数、错误次数、业务计数 |
| **Metrics Timer** | 耗时统计 | 响应时间分布（P50/P90/P99） | 方法执行时间、接口响应时间 |
| **Metrics Summary** | 数值分布 | 数值分布统计 | 请求大小、响应大小 |
| **Metrics Gauge** | 瞬时值 | 当前值 | 线程池状态、队列大小、缓存大小 |
### 核心差异
- **CAT Transaction** vs **Metrics Timer**
  - Transaction：提供完整的调用链路追踪（LogView）+ 异常堆栈 + 响应时间统计
  - Timer：专注于响应时间分布统计，轻量级
  - **推荐**：核心链路用 Transaction，性能统计用 Timer
- **CAT Event** vs **Metrics Counter**
  - Event：可查看具体的采样调用链路，支持失败率统计
  - Counter：纯粹的累计计数，性能更好
  - **推荐**：异常告警用 Event，常规计数用 Counter
---
## CAT Transaction 使用指南
### 标准使用模式
#### ✅ 完整模式（推荐）
```java
Transaction transaction = Cat.newTransaction(type, name);
try {
    // 执行业务逻辑
    result = doSomething();
    
    // 成功时设置状态
    transaction.setSuccessStatus();  // 等同于 setStatus("0")
    
    // 可选：添加业务数据
    transaction.addData("userId", userId);
    transaction.addData("count", count);
    
} catch (Exception e) {
    // 失败时记录异常
    transaction.setStatus(e);  // 记录异常类型
    Cat.logError(e);          // 上报异常详情到 Error 报表
    throw e;
} finally {
    // 必须调用 complete() 完成监控
    transaction.complete();
}
```
#### ⚡ 简化模式（仅记录执行次数）
```java
// 适用于不关心成功/失败，只记录调用次数的场景
Transaction t = Cat.newTransaction("BizSolution.Goods", "QueryCache");
try {
    result = queryFromCache();
} finally {
    t.complete();
}
```
### Type/Name 命名规范
基于项目实践总结的命名模式：
| Type 格式 | Name 格式 | 示例 | 说明 |
|-----------|-----------|------|------|
| `Biz.{Identifier}` | `{Category}.{MethodName}` | `Biz.Goods` / `RECALL.doProcess` | 通用业务方法 |
| `BizSolution.{SolutionKey}` | `{Category}` | `BizSolution.Goods` / `RECALL` | Solution 层 |
| `RPC.{ServiceName}` | `{MethodName}` | `RPC.ProductClient` / `getProductInfo` | RPC 调用 |
| `Cache.{CacheName}` | `{Operation}` | `Cache.Redis` / `GET` | 缓存操作 |
| `DB.{TableName}` | `{Operation}` | `DB.User` / `SELECT` | 数据库操作 |
#### 命名最佳实践
```java
// ✅ 推荐：清晰的业务含义
Cat.newTransaction("BizSolution.Goods", "RECALL");
Cat.newTransaction("Biz.Goods", "RefreshGoodsPool");
Cat.newTransaction("RPC.ProductClient", "batchQueryProductInfo");
// ❌ 不推荐：过于笼统
Cat.newTransaction("Business", "method");
Cat.newTransaction("Service", "call");
```
### 状态设置规则
```java
// ✅ 成功状态
transaction.setSuccessStatus();  // 推荐
transaction.setStatus("0");      // 等同于上面
// ❌ 失败状态（任意非"0"字符串）
transaction.setStatus("failed");         // 字符串标记
transaction.setStatus(e);                // 异常对象（推荐）
transaction.setStatus(e.getClass().getSimpleName()); // 异常类名
```
### TP90/TP99 统计准确性
根据 CAT 文档：
| 维度 | 准确性 | 说明 |
|------|--------|------|
| 小时 + 单机 | ✅ 完全准确 | 基于全量数据分桶统计 |
| 小时 + 集群 ALL | ⚠️ 近似值 | 加权平均，波动大时误差大 |
| 天/周/月 | ⚠️ 近似值 | 加权合并，仅供参考 |
**告警建议**：
- 使用 **单机级别** 的 TP90/TP99 设置告警阈值
- 集群 ALL 的 TP 值仅供趋势观察
### 性能优化要点
#### 1. 缓存元数据（避免运行时反射）
```java
// 参考 MonitorAspect 的实现
private final ConcurrentHashMap<String, MetaData> metaDataCache = new ConcurrentHashMap<>();
private MetaData getOrCreateMetaData(String cacheKey) {
    return metaDataCache.computeIfAbsent(cacheKey, key -> {
        // 一次性完成所有反射操作
        return buildMetaData();
    });
}
```
#### 2. 配置化开关（避免无效创建）
```java
Transaction transaction = null;
if (monitorConfig.isTransactionEnabled()) {
    transaction = Cat.newTransaction(type, name);
}
try {
    // 业务逻辑
} finally {
    if (transaction != null) {
        transaction.complete();
    }
}
```
#### 3. 采样控制（高频场景）
```java
// 对于 QPS 超过 1000 的接口，可以采样
if (shouldSample(requestId)) {
    Transaction t = Cat.newTransaction(type, name);
    // ...
}
private boolean shouldSample(String requestId) {
    // 1% 采样
    return Math.abs(requestId.hashCode()) % 100 == 0;
}
```
---
## CAT Event 使用指南
### 使用场景
| 场景 | 示例 | 说明 |
|------|------|------|
| 条件分支统计 | 不同推荐策略的命中次数 | 记录各分支执行频率 |
| 业务状态变更 | 商品上架/下架、订单状态流转 | 追踪业务状态变化 |
| 异常情况告警 | 库存不足、配置缺失、黑名单拦截 | 快速发现异常 |
| 降级/熔断 | 缓存降级、限流触发 | 监控系统保护措施 |
### 标准使用模式
#### 模式 1：简单事件记录（只记录次数）
```java
// 记录商品上架事件
Cat.logEvent("Goods", "Item.OnSale");
// 记录黑名单过滤
Cat.logEvent("Goods.Filter", "Blacklist");
// 记录降级
Cat.logEvent("Goods", "CacheFallback");
```
#### 模式 2：带状态的事件记录
```java
Event event = Cat.newEvent("Goods", "QueryInventory");
try {
    int inventory = queryInventory(itemId);
    if (inventory > 0) {
        event.setSuccessStatus();
    } else {
        event.setStatus("NoInventory");
    }
} catch (Exception e) {
    event.setStatus(e);
    Cat.logError(e);
} finally {
    event.complete();
}
```
#### 模式 3：带数据的事件记录
```java
// 记录具体的业务数据
String data = String.format("itemId=%s,count=%d", itemId, count);
Cat.logEvent("Goods", "CandidateAdd", "0", data);
// 失败事件（status="1"）
Cat.logEvent("Goods", "InventoryCheck", "1", "itemId=" + itemId);
```
### Event + Metrics 组合模式（慢方法检测）
参考 `MonitorUtils.recordSlowMethod` 的实现：
```java
public static void recordSlowMethod(String transactionName, String customName, 
                                   Category category, long durationMicroSeconds,
                                   Map<String, String> customTags) {
    long durationMs = durationMicroSeconds / 1000L;
    
    try {
        // 1. 记录通用慢方法 Event（便于全局查看）
        Map<String, String> eventData = new HashMap<>(customTags);
        eventData.put("duration", durationMs + "ms");
        String data = buildData(eventData);
        Cat.logEvent("SlowMethod", transactionName, "1", data);
        
        // 2. 记录具体业务慢方法 Event（便于业务维度查看）
        String eventName = category.getNormalizeValue() + ".SlowMethod";
        Cat.logEvent(transactionName, eventName, "1", data);
        
        // 3. 同时上报 Metrics Timer（用于聚合统计和可视化）
        MetricsUtils.timer("app_slow_method_timer_us")
            .addTag("env", EnvPropertiesUtils.getEnv())
            .addTag("name", customName)
            .addTag("category", category.getValue())
            .record(durationMicroSeconds, TimeUnit.MICROSECONDS);
            
    } catch (Exception e) {
        log.error("Slow method record failed", e);
    }
}
```
---
## Metrics 使用指南
### 依赖引入
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>infra-framework-monitor-base</artifactId>
</dependency>
```
### 1. Counter（计数器）
#### 基础用法
```java
// 简单计数
MetricsUtils.counter("goods_recall_total").increment();
// 指定增量
MetricsUtils.counter("goods_items_total").increment(100);
// 带标签的计数
MetricsUtils.counter("goods_filter_total")
    .addTag("env", "prod")
    .addTag("reason", "blacklist")
    .increment();
```
#### 项目实践案例
```java
// 监控召回数量
MetricsUtils.counter("app_output_count_total")
    .addTag("env", env)
    .addTag("name", "Goods")
    .addTag("category", "RECALL")
    .increment(outputCount);
// 监控过滤数量
MetricsUtils.counter("app_filter_count_total")
    .addTag("env", env)
    .addTag("name", "Goods")
    .addTag("category", "FILL")
    .addTag("filter_reason", "price_too_high")
    .increment(filterCount);
// 监控商品池大小（定时任务）
MetricsUtils.counter("goods_pool_size")
    .addTag("env", env)
    .addTag("type", "goods_pool")
    .increment(poolSize);
```
### 2. Timer（计时器）
#### 基础用法
```java
// 方式1：使用 Stopwatch
Stopwatch stopwatch = Stopwatch.createStarted();
try {
    // 业务逻辑
    result = doSomething();
} finally {
    MetricsUtils.timer("goods_refresh_timer")
        .addTag("stage", "build_pool")
        .record(stopwatch.elapsed());
}
// 方式2：使用纳秒时间戳
long startTime = System.nanoTime();
try {
    // 业务逻辑
    result = doSomething();
} finally {
    long duration = System.nanoTime() - startTime;
    MetricsUtils.timer("goods_rpc_timer")
        .addTag("client", "ProductClient")
        .addTag("method", "batchQuery")
        .record(duration, TimeUnit.NANOSECONDS);
}
```
#### 带异常标签的 Timer
```java
TimerBuilder timer = MetricsUtils.timer("goods_query_timer")
    .addTag("operation", "queryInventory");
Stopwatch stopwatch = Stopwatch.createStarted();
try {
    result = inventoryClient.query(itemId);
    timer.addTag("result", "success");
} catch (TimeoutException e) {
    timer.addTag("result", "timeout");
    throw e;
} catch (Exception e) {
    timer.addTag("result", e.getClass().getSimpleName());
    throw e;
} finally {
    timer.record(stopwatch.elapsed());
}
```
### 3. Summary（摘要）
用于记录数值分布，如批次大小、数据量等。
```java
// 记录批次大小
MetricsUtils.summary("goods_batch_size")
    .addTag("operation", "rpc_query")
    .record(batchSize);
// 记录候选池大小变化
MetricsUtils.summary("goods_candidate_delta")
    .addTag("type", "add")
    .record(addedCount);
```
### 4. Gauge（仪表盘）
用于记录瞬时值，如缓存大小、队列长度等。
```java
// 动态监控商品池大小
MetricsUtils.gauge("goods_pool_gauge")
    .addTag("type", "goods_pool")
    .build(() -> redisDao.getGoodsPoolSize());
// 静态值设置
MetricsUtils.gauge("goods_config_gauge")
    .addTag("key", "min_candidate_size")
    .build()
    .setValue(100L);
```
### Metrics 命名规范
```java
// ✅ 推荐：项目前缀 + 业务含义 + 指标类型
"app_output_count_total"          // Counter
"app_slow_method_timer_us"        // Timer（微秒）
"goods_pool_size"            // Gauge
"goods_batch_size_summary"   // Summary
// ❌ 不推荐：缺少上下文
"count"
"timer"
"size"
```
---
## 组合使用模式
### 模式 1：Transaction + Metrics（核心业务链路）
```java
public void refreshGoodsPool() {
    // CAT Transaction：记录完整链路 + 异常
    Transaction t = Cat.newTransaction("BizSolution.Goods", "RefreshPool");
    
    // Metrics Timer：记录耗时分布
    Stopwatch stopwatch = Stopwatch.createStarted();
    
    try {
        // 业务逻辑
        int addedCount = buildGoodsPool();
        int removedCount = removeExpiredGoods();
        
        // 添加业务数据到 Transaction
        t.addData("added", addedCount);
        t.addData("removed", removedCount);
        t.setSuccessStatus();
        
        // Metrics 记录数量
        MetricsUtils.counter("goods_pool_refresh_total")
            .addTag("result", "success")
            .increment();
        MetricsUtils.counter("goods_pool_added_total")
            .increment(addedCount);
        MetricsUtils.counter("goods_pool_removed_total")
            .increment(removedCount);
        
    } catch (Exception e) {
        t.setStatus(e);
        Cat.logError(e);
        
        MetricsUtils.counter("goods_pool_refresh_total")
            .addTag("result", "failed")
            .addTag("error", e.getClass().getSimpleName())
            .increment();
        
        throw e;
    } finally {
        t.complete();
        
        // Metrics Timer 记录耗时
        MetricsUtils.timer("goods_pool_refresh_timer")
            .record(stopwatch.elapsed());
    }
}
```
### 模式 2：Event + Counter（业务事件）
```java
public void handleItemChange(ItemChangeEvent event) {
    if (event.isOnSale()) {
        // CAT Event：采样调用链路
        Cat.logEvent("Goods", "Item.OnSale");
        
        // Metrics Counter：累计计数
        MetricsUtils.counter("goods_item_change_total")
            .addTag("action", "on_sale")
            .increment();
            
    } else {
        Cat.logEvent("Goods", "Item.OffSale");
        
        MetricsUtils.counter("goods_item_change_total")
            .addTag("action", "off_sale")
            .increment();
    }
}
```
### 模式 3：慢方法检测三件套
```java
// 在业务代码中
long startTime = System.nanoTime();
try {
    result = doSlowOperation();
} finally {
    long durationUs = (System.nanoTime() - startTime) / 1000;
    long durationMs = durationUs / 1000;
    
    // 慢方法阈值：500ms
    if (durationMs > 500) {
        MonitorUtils.recordSlowMethod(
            "BizSolution.Goods",
            "Goods",
            Category.BIZ,
            durationUs,
            Map.of("operation", "buildPool")
        );
    }
}
```
`MonitorUtils.recordSlowMethod` 内部实现：
```java
// 1. Event（全局慢方法）
Cat.logEvent("SlowMethod", transactionName, "1", "duration=500ms");
// 2. Event（业务维度）
Cat.logEvent(transactionName, "BIZ.SlowMethod", "1", "duration=500ms");
// 3. Timer（聚合统计）
MetricsUtils.timer("app_slow_method_timer_us")
    .addTag("category", "BIZ")
    .addTag("name", "Goods")
    .record(500000, TimeUnit.MICROSECONDS);
```
---
## 项目最佳实践
### 1. 统一的监控工具类
项目提供了 `MonitorUtils` 类，封装常用监控逻辑：
```java
// 记录召回阶段指标（无 inputCount）
MonitorUtils.recordRecallMetrics(
    Category.RECALL, 
    "Goods", 
    outputCount, 
    customTags
);
// 记录填充阶段指标（有 inputCount）
MonitorUtils.recordMetrics(
    Category.FILL, 
    "Goods", 
    inputCount, 
    outputCount, 
    customTags
);
// 记录慢方法
MonitorUtils.recordSlowMethod(
    "BizSolution.Goods",
    "Goods",
    Category.BIZ,
    durationMicroSeconds,
    customTags
);
```
### 2. 常量管理
所有监控指标名称和标签定义在常量类中：
```java
public class GoodsMetricsConstant {
    // 指标名称
    public static final String POOL_REFRESH_TOTAL = "goods_pool_refresh_total";
    public static final String ITEM_CHANGE_TOTAL = "goods_item_change_total";
    public static final String RECALL_TOTAL = "goods_recall_total";
    
    // 标签名称
    public static final String TAG_ACTION = "action";
    public static final String TAG_RESULT = "result";
    public static final String TAG_CATEGORY = "category";
    public static final String TAG_FILTER_REASON = "filter_reason";
}
```
### 3. 异常处理模式
监控代码应该包装在 try-catch 中，避免影响业务逻辑：
```java
// ✅ 推荐：监控异常不影响业务
try {
    MetricsUtils.counter("goods_pool_size")
        .addTag("type", "goods_pool")
        .increment(poolSize);
} catch (Exception e) {
    log.warn("Metrics log failed: {}", e.getMessage());
    // 业务继续执行
}
// ❌ 不推荐：监控异常中断业务
MetricsUtils.counter("goods_pool_size")
    .increment(poolSize);  // 可能抛异常
```
### 4. 标签值规范化
对于数值类型的标签，建议进行范围化处理，避免高基数：
```java
private String getCountRange(int count) {
    if (count == 0) return "0";
    if (count <= 10) return "1-10";
    if (count <= 50) return "11-50";
    if (count <= 100) return "51-100";
    if (count <= 500) return "101-500";
    return "500+";
}
// 使用
MetricsUtils.counter("goods_pool_size")
    .addTag("size_range", getCountRange(poolSize))
    .increment();
```
### 5. 环境标签统一
所有 Metrics 都应该带上环境标签：
```java
String env = EnvPropertiesUtils.getEnv();  // prod/beta/test
MetricsUtils.counter("goods_recall_total")
    .addTag("env", env)  // 统一添加环境标签
    .addTag("category", "RECALL")
    .increment();
```
---
## 监控设计原则
### 1. 业务相关性
监控指标应该反映真实的业务状态，而不是技术指标。
```java
// ✅ 推荐：业务指标
"goods_recall_count"          // 召回商品数量
"goods_filter_blacklist"      // 黑名单过滤次数
"goods_inventory_check_fail"  // 库存检查失败次数
// ❌ 不推荐：纯技术指标
"method_call_count"
"loop_iterations"
```
### 2. 可操作性
指标异常时应该有明确的处理方案。
```java
// ✅ 可操作：能明确定位问题
"goods_candidate_size"  // 候选池过小 → 检查 MQ 消费
"goods_rpc_timeout"     // RPC 超时 → 检查下游服务
"goods_filter_rate"     // 过滤率过高 → 检查过滤规则
// ❌ 不可操作：无法定位问题
"system_health"
"business_status"
```
### 3. 层次化
从全局到局部，从粗粒度到细粒度。
```java
// 第一层：全局统计
Cat.newTransaction("BizSolution.Goods", "TOTAL");
// 第二层：阶段统计
Cat.newTransaction("BizSolution.Goods", "RECALL");
Cat.newTransaction("BizSolution.Goods", "FILL");
// 第三层：操作统计
Cat.logEvent("Goods.Recall", "QueryCache");
Cat.logEvent("Goods.Recall", "FilterBlacklist");
```
### 4. 标准化
同类型的指标使用统一的命名和标签规范。
```java
// ✅ 标准化：统一的前缀和后缀
"app_output_count_total"     // Recall 输出
"app_input_count_total"      // Fill 输入
"app_filter_count_total"     // Fill 过滤
// ✅ 标准化：统一的标签名
.addTag("env", env)              // 所有指标都有
.addTag("category", category)    // 所有业务指标都有
.addTag("name", name)            // 所有业务指标都有
```
---
## 商品功能监控设计方案
### B 端数据构建层
#### 1. MQ 消费者
```java
public class GoodsCandidateConsumer {
    
    public void onMessage(ItemChangeEvent event) {
        Transaction t = Cat.newTransaction("BizSolution.Goods", "MQ.ItemChange");
        Stopwatch stopwatch = Stopwatch.createStarted();
        
        try {
            // 处理消息
            candidateManager.handleItemChange(event);
            t.setSuccessStatus();
            
            // Event 记录商品上下架
            if (event.isOnSale()) {
                Cat.logEvent("Goods", "Item.OnSale");
                MetricsUtils.counter("goods_item_change_total")
                    .addTag("action", "on_sale")
                    .increment();
            } else {
                Cat.logEvent("Goods", "Item.OffSale");
                MetricsUtils.counter("goods_item_change_total")
                    .addTag("action", "off_sale")
                    .increment();
            }
            
        } catch (Exception e) {
            t.setStatus(e);
            Cat.logError(e);
            
            MetricsUtils.counter("goods_mq_consume_total")
                .addTag("result", "failed")
                .addTag("error", e.getClass().getSimpleName())
                .increment();
            
            throw e;
        } finally {
            t.complete();
            
            MetricsUtils.timer("goods_mq_consume_timer")
                .record(stopwatch.elapsed());
        }
    }
}
```
#### 2. 候选池管理器
```java
public class GoodsCandidateManager {
    
    public void handleItemChange(ItemChangeEvent event) {
        String itemId = event.getItemId();
        
        // 检查是否为供销商品
        if (!event.isSupply()) {
            Cat.logEvent("Goods.Filter", "NotSupply");
            MetricsUtils.counter("goods_filter_total")
                .addTag("reason", "not_supply")
                .increment();
            return;
        }
        
        // 检查库存
        int inventory = inventoryClient.getItemInventory(itemId);
        if (inventory <= 0) {
            Cat.logEvent("Goods.Filter", "NoInventory");
            MetricsUtils.counter("goods_filter_total")
                .addTag("reason", "no_inventory")
                .increment();
            return;
        }
        
        // 添加到候选池
        redisDao.addToCandidate(itemId);
        Cat.logEvent("Goods.Candidate", "AddSuccess");
        MetricsUtils.counter("goods_candidate_add_total")
            .increment();
    }
}
```
#### 3. 定时任务
```java
@Scheduled(cron = "0 */1 * * * ?")  // 每分钟执行
public class GoodsRefreshTask {
    
    public void execute() {
        Transaction t = Cat.newTransaction("BizSolution.Goods", "Schedule.Refresh");
        Stopwatch stopwatch = Stopwatch.createStarted();
        
        try {
            // 1. 刷新商品池
            int addedCount = goodsPoolBuilder.refresh();
            
            // 2. 构建排序ID
            sortedIdBuilder.build();
            
            // 3. 构建Tab配置
            tabConfigBuilder.build();
            
            // 添加业务数据
            t.addData("added", addedCount);
            t.setSuccessStatus();
            
            // Metrics 记录
            MetricsUtils.counter("goods_refresh_total")
                .addTag("result", "success")
                .increment();
            
        } catch (Exception e) {
            t.setStatus(e);
            Cat.logError(e);
            
            MetricsUtils.counter("goods_refresh_total")
                .addTag("result", "failed")
                .increment();
            
            throw e;
        } finally {
            t.complete();
            
            MetricsUtils.timer("goods_refresh_timer")
                .record(stopwatch.elapsed());
        }
    }
}
```
#### 4. 商品池构建
```java
public class GoodsPoolBuilder {
    
    public int refresh() {
        Transaction t = Cat.newTransaction("Biz.Goods", "BIZ.BuildPool");
        
        try {
            // 获取候选池
            Set<String> candidateIds = redisDao.getCandidateIds();
            
            // Metrics 记录候选池大小
            int candidateSize = candidateIds.size();
            MetricsUtils.gauge("goods_candidate_size")
                .build()
                .setValue((long) candidateSize);
            
            // 批量查询商品信息
            List<GoodsInfo> goodsList = queryGoodsInfo(candidateIds);
            
            // 黑名单过滤
            int beforeSize = goodsList.size();
            goodsList = filterBlacklist(goodsList);
            int blacklistCount = beforeSize - goodsList.size();
            
            if (blacklistCount > 0) {
                Cat.logEvent("Goods.Filter", "Blacklist");
                MetricsUtils.counter("goods_filter_total")
                    .addTag("reason", "blacklist")
                    .increment(blacklistCount);
            }
            
            // Diff 更新商品池
            DiffResult diff = diffAndUpdate(goodsList);
            
            // 记录更新详情
            Cat.logEvent("Goods.Pool", "Add", "0", diff.addedCount + " items");
            Cat.logEvent("Goods.Pool", "Remove", "0", diff.removedCount + " items");
            
            MetricsUtils.counter("goods_pool_update_total")
                .addTag("action", "add")
                .increment(diff.addedCount);
            MetricsUtils.counter("goods_pool_update_total")
                .addTag("action", "remove")
                .increment(diff.removedCount);
            
            // Metrics 记录商品池大小
            int poolSize = redisDao.getGoodsPoolSize();
            MetricsUtils.gauge("goods_pool_size")
                .build()
                .setValue((long) poolSize);
            
            t.setSuccessStatus();
            return diff.addedCount;
            
        } catch (Exception e) {
            t.setStatus(e);
            Cat.logError(e);
            throw e;
        } finally {
            t.complete();
        }
    }
}
```
### C 端召回层
```java
public class GoodsRecall extends BaseRecall {
    
    @Override
    public DataList doProcess(ProcContext ctx) {
        Transaction t = Cat.newTransaction("BizSolution.Goods", "RECALL");
        Stopwatch stopwatch = Stopwatch.createStarted();
        
        try {
            // 从 Redis 召回排序后的 itemIds
            Long categoryId = ctx.getParam("categoryId");
            List<String> itemIds = redisDao.getSortedItemIds(categoryId);
            
            // 添加数据
            t.addData("categoryId", categoryId);
            t.addData("count", itemIds.size());
            t.setSuccessStatus();
            
            // Metrics 记录召回数量
            MetricsUtils.counter("app_output_count_total")
                .addTag("env", EnvPropertiesUtils.getEnv())
                .addTag("name", "Goods")
                .addTag("category", "RECALL")
                .increment(itemIds.size());
            
            return DataList.of(itemIds);
            
        } catch (Exception e) {
            t.setStatus(e);
            Cat.logError(e);
            throw e;
        } finally {
            t.complete();
            
            MetricsUtils.timer("goods_recall_timer")
                .record(stopwatch.elapsed());
        }
    }
}
```
### C 端 Fill 层
```java
public class GoodsFill extends BaseFill {
    
    @Override
    public DataList doProcess(ProcContext ctx, DataList dataList) {
        Transaction t = Cat.newTransaction("BizSolution.Goods", "FILL");
        Stopwatch stopwatch = Stopwatch.createStarted();
        
        try {
            int inputCount = dataList.size();
            
            // 查询商品详情（动态查询最低价 SKU）
            List<GoodsDetail> details = queryGoodsDetail(dataList);
            
            // 过滤
            int beforeFilter = details.size();
            details = filterByPrice(details);
            int filterCount = beforeFilter - details.size();
            
            // 记录过滤
            if (filterCount > 0) {
                Cat.logEvent("Goods.Fill", "FilterByPrice");
                MetricsUtils.counter("app_filter_count_total")
                    .addTag("env", EnvPropertiesUtils.getEnv())
                    .addTag("name", "Goods")
                    .addTag("category", "FILL")
                    .addTag("filter_reason", "price_too_high")
                    .increment(filterCount);
            }
            
            int outputCount = details.size();
            
            // Metrics 记录输入输出
            String env = EnvPropertiesUtils.getEnv();
            MetricsUtils.counter("app_input_count_total")
                .addTag("env", env)
                .addTag("name", "Goods")
                .addTag("category", "FILL")
                .increment(inputCount);
            MetricsUtils.counter("app_output_count_total")
                .addTag("env", env)
                .addTag("name", "Goods")
                .addTag("category", "FILL")
                .increment(outputCount);
            
            t.setSuccessStatus();
            return DataList.of(details);
            
        } catch (Exception e) {
            t.setStatus(e);
            Cat.logError(e);
            throw e;
        } finally {
            t.complete();
            
            MetricsUtils.timer("goods_fill_timer")
                .record(stopwatch.elapsed());
        }
    }
}
```
---
## 常见问题
### Q1: Transaction 一定要 complete() 吗？
**A**: 是的，必须在 finally 块中调用 `complete()`，否则会导致：
1. Transaction 数据不会上报
2. CAT 客户端内存泄漏
3. 调用链路追踪不完整
```java
// ❌ 错误：缺少 finally
Transaction t = Cat.newTransaction(type, name);
result = doSomething();
t.complete();  // 如果抛异常，永远不会执行
// ✅ 正确
Transaction t = Cat.newTransaction(type, name);
try {
    result = doSomething();
} finally {
    t.complete();  // 保证一定执行
}
```
### Q2: 什么时候用 Transaction，什么时候用 Event？
**A**: 
- **Transaction**：关注执行时间、需要调用链路、需要异常堆栈的场景
  - 核心业务方法
  - RPC 调用
  - DB 操作
  
- **Event**：只关注发生次数、不关注时间的场景
  - 业务状态变更
  - 条件分支统计
  - 降级/熔断触发
### Q3: CAT 和 Metrics 有什么区别，应该用哪个？
**A**: 
- **CAT Transaction**：
  - ✅ 优势：调用链路追踪、异常堆栈、TP90/TP99 统计
  - ❌ 劣势：性能开销较大、只能在 CAT 平台查看
  
- **Metrics Timer**：
  - ✅ 优势：轻量级、可以导出到多种监控平台（Prometheus/Grafana）
  - ❌ 劣势：无调用链路、无异常堆栈
  
**推荐组合使用**：
```java
// Transaction 记录完整链路
Transaction t = Cat.newTransaction(type, name);
try {
    result = doSomething();
    t.setSuccessStatus();
} catch (Exception e) {
    t.setStatus(e);
    Cat.logError(e);
} finally {
    t.complete();
}
// Metrics 记录性能分布（用于 Grafana 可视化）
MetricsUtils.timer("operation_timer")
    .record(duration);
```
### Q4: 为什么不直接使用 micrometer API？
**A**: 直接使用 micrometer 容易导致内存泄漏，特别是在动态标签场景下。`MetricsUtils` 提供了：
1. 自动清理机制（30 分钟未使用的 Summary/Timer 自动清理）
2. 内存保护（每个 namespace 最多 3W 个 Meter）
3. 更简洁的 API（链式调用）
### Q5: 标签过多会有什么问题？
**A**: 标签的笛卡尔积会产生大量的时间序列，例如：
- 3 个标签，每个标签 10 个值 → 1000 个时间序列
- 5 个标签，每个标签 10 个值 → 100,000 个时间序列
**建议**：
1. 控制标签数量（一般不超过 5 个）
2. 避免高基数标签（用户ID、时间戳等）
3. 对数值标签进行范围化处理
```java
// ❌ 不推荐：高基数
MetricsUtils.counter("api_request")
    .addTag("user_id", userId)  // 百万级基数
    .increment();
// ✅ 推荐：低基数
MetricsUtils.counter("api_request")
    .addTag("user_type", getUserType(userId))  // 几个固定值
    .increment();
```
### Q6: TP90/TP99 的统计准确吗？
**A**: 根据 CAT 文档：
- ✅ **准确**：小时粒度 + 单机维度
- ⚠️ **近似**：小时粒度 + 集群 ALL 维度（加权平均，波动大时误差大）
- ⚠️ **近似**：天/周/月级别（加权合并，仅供参考）
**告警建议**：使用单机级别的 TP90/TP99 设置告警阈值。
### Q7: Cat.logMetricForCount 参数有限制吗？
**A**: 有的，部分版本只支持 1-2 个参数：
```java
// 部分版本支持
Cat.logMetricForCount("metric_name");
Cat.logMetricForCount("metric_name", count);
// ❌ 部分版本不支持多参数
Cat.logMetricForCount("metric_name", "tag1", value1, "tag2", value2);
```
**建议**：使用 `MetricsUtils.counter()` 替代，功能更强大。
### Q8: 监控代码异常会影响业务吗？
**A**: 如果不做异常处理，会影响业务。**必须**使用 try-catch 包装：
```java
// ✅ 推荐：监控异常不影响业务
try {
    MetricsUtils.counter("business_count").increment();
    Cat.logEvent("Business", "Event");
} catch (Exception e) {
    log.warn("Monitor failed: {}", e.getMessage());
    // 业务继续执行
}
```
### Q9: 采样会影响统计准确性吗？
**A**: 
- **Transaction 采样**：会影响 QPS、错误率、TP99 等统计（按采样率换算）
- **Event 采样**：会影响次数统计
- **Metrics 不采样**：始终全量统计，准确
**建议**：
- 低频接口（QPS < 100）：不采样
- 高频接口（QPS > 1000）：可以 1%-10% 采样
- 核心链路：不采样
- Metrics 始终全量统计
---
