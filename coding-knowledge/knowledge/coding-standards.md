---
name: 编码规范
stages:
  - coding
source: base
priority: high
---
# 编码规范
> Agent 生成代码时的核心规则清单。每条规则力求一句话说清，详细示例和模板通过索引链接到专属文档。
---
## 1. 命名规范
### 类命名
| 类型 | 格式 | 示例 |
|------|------|------|
| Manager | `{业务}Manager` | `SkuInfoManager` |
| Client | `{服务}Client` | `ProductClient` |
| Converter | `{对象}Converter` | `MixCardContentConverter` |
| BO/Entity | `{业务}Bo` / `{业务}Entity` | `MallHomeFeedBo`, `GoodsEntity` |
| Enum | `{名称}Enum` | `FeedTypeEnum` |
| 常量 | `{名称}Constant` | `RedisKeyConstant`, `XrayConstant` |
| 配置 | `{功能}Config` | `ApolloConfig` |
| 异常 | `{模块}Exception` | `UserBizException` |
### 方法命名
| 类型 | 格式 | 示例 |
|------|------|------|
| 查询 | `get{X}` / `query{X}` | `getSkuInfo()` |
| 批量查询 | `batchGet{X}` / `list{X}` | `batchGetSaleSkus()` |
| 异步 | `async{Method}` | `asyncGetSkuInfo()` |
| 转换 | `convert{A}To{B}` | `convertToThrift()` |
| 构建 | `build{X}` | `buildRedKey()` |
| 校验 | `check{X}` / `validate{X}` | `checkBatchCreateReq()` |
### 变量命名
- **禁止**无意义命名：`temp`, `a`, `list1`, `data`
- 使用有业务含义的名称：`validUserIds`, `skuInfoMap`, `processedData`
---
## 2. 目录结构与分层
```
src/main/java/com/example/{module}/
├── common/              # 常量(constant/)、枚举(enums/)、工具(util/)
├── config/              # Apollo 配置、异步线程池配置
├── client/              # 外部 RPC 调用封装
├── manager/             # 复杂业务逻辑、多服务协调
├── service/             # 单一职责业务逻辑
├── converter/           # Entity ↔ Thrift 互转
├── entity/              # BO/DTO/Entity
├── mq/                  # Kafka/RocketMQ
├── dao/                 # 数据访问层
└── exceptions/          # 自定义异常
```
| 层级 | 职责 |
|------|------|
| **Manager** | 复杂业务逻辑、多服务协调 |
| **Client** | RPC 调用封装、降级处理 |
| **Converter** | Entity ↔ Thrift 互转 |
| **Service** | 单一职责业务逻辑 |
---
## 3. RPC 调用
> 详见：[rpc-integration-guide.md](rpc-integration-guide.md)
### 规则
1. **推荐使用 `RpcCallUtil.execute()` 统一封装**
2. 使用 `ContextHelper.getContext()` 传递上下文
3. 弱依赖启用降级（`enableDegrade=true`），强依赖不降级
4. **必须设置超时**：强依赖 1500-2000ms，弱依赖 1000ms
5. 批量查询必须分批：`Lists.partition(ids, BATCH_SIZE)` + 并发
### 最小示例
```java
return RpcCallUtil.execute(
    "ProductClient",                          // type
    "productResourceService.getResourceConfig", // method
    () -> productResourceService.getResourceConfig(ContextHelper.getContext(), request),
    null, false, false, null, true, request  // baseException, logResult, enableDegrade, degradeResult, checkResponseSuccess, args
);
```
**RpcCallUtil 参数速查**：
| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `type` | 客户端名称 | `{Service}Client` |
| `method` | 方法全名 | `service.method` |
| `supplier` | 调用逻辑 | Lambda |
| `baseException` | 自定义异常 | 业务异常或 `null` |
| `logResult` | 记录响应 | 核心 `true`，其他 `false` |
| `enableDegrade` | 启用降级 | 弱依赖 `true` |
| `degradeResult` | 降级返回值 | 默认值或 `null` |
| `checkResponseSuccess` | 检查响应 | Thrift `true` |
---
## 4. 异步并发
> 详见：[async-concurrency-guide.md](async-concurrency-guide.md)
### 规则
1. **禁止** `new Thread()`，使用项目线程池：`businessTaskExecutor`、`engineTaskExecutor`、`traceExecutor`
2. 多个独立 RPC 并行 → `FutureUtil.invokeTask()` + `FutureUtil.getWithDefault()`
3. 批量元素同类操作并行 → `GroupExecuteUtil.execute()`
4. 大量 ID 分批并发查询 → `Lists.partition()` + `businessTaskExecutor.submit()` + `FutureTaskHandler.getWithDefault()`
5. **所有 Future 必须设置超时**，禁止无限等待
6. ThreadLocal 必须 `try-finally { threadLocal.remove() }`
---
## 5. 异常处理
### 场景分类（按场景选择，不要一刀切 try-catch）
| 场景 | 处理方式 | 说明                                                    |
|------|---------|-------------------------------------------------------|
| **RPC 调用** | `RpcCallUtil` 统一处理 | 无需手动 try-catch                                        |
| **参数验证/类型转换** | try-catch 返回默认值 | 如 `Long.parseLong()`                                  |
| **批量操作** | 异常隔离 | 单个批次失败不影响整体                                           |
| **监控打点** | try-catch 保护 | 避免监控异常影响业务                                            |
| **业务规则不满足** | 直接抛出业务异常 | `throw new UserBizException(errorCode, msg)` |
### 基本原则
- **严禁**吞异常：禁止 `catch (Exception e) {}` 或 `e.printStackTrace()`
- 异常日志必须包含上下文信息（userId、itemId 等）
- 核心链路失败时返回缓存/降级数据，不直接抛 500
### 业务异常体系
```java
// 参数校验失败 → 直接抛出
if (StringUtils.isAnyBlank(userId, itemId)) {
    throw new UserBizException(UserErrorCode.INVALID_PARAM_ERROR, "参数不能为空");
}
```
---
## 6. 监控打点
> 详见：[cat-and-metrics-guide.md](cat-and-metrics-guide.md)
### CAT 规则
| 类型 | 使用场景 | 模板 |
|------|---------|------|
| **Transaction** | 核心业务链路、RPC 调用 | `Cat.newTransaction(type, name)` → `setSuccessStatus()` / `setStatus(e)` → `complete()` |
| **Event** | 降级记录、缓存命中率、业务分支 | `Cat.logEvent(type, name)` |
**Type/Name 命名**：`BizSolution.{Key}` / `RPC.{Service}` / `Cache.{Name}`
### Metrics 规则
| 类型 | 使用场景 | 模板 |
|------|---------|------|
| **Counter** | 计数（请求次数、错误次数） | `MetricsUtils.counter("name").addTag(k,v).increment()` |
| **Timer** | 耗时（方法执行、接口响应） | `MetricsUtils.timer("name").addTag(k,v).record(elapsed)` |
| **Gauge** | 状态（队列大小、池大小） | `MetricsUtils.gauge("name").addTag(k,v).build(supplier)` |
### 关键要求
1. **监控代码必须 try-catch 包裹**，不影响业务
2. 数值标签做范围化处理（避免高基数）
3. 所有 Metrics 带环境标签 `addTag("env", EnvPropertiesUtils.getEnv())`
4. 指标名、标签名定义在常量类中（`XrayConstant`、`MetricsConstant`）
---
## 7. 日志规范
### 格式
```
log.info("[ClassName.methodName] req: {}", JSON.toJSONString(request));
log.info("[ClassName.methodName] 业务描述. userId = {}, skuId = {}", userId, skuId);
log.error("[ClassName.methodName][ERROR] 错误描述", exception);
```
### 级别
| 级别 | 场景 |
|------|------|
| `ERROR` | 需人工介入的系统错误（DB 连不上） |
| `WARN` | 预期异常（参数校验不通过） |
| `INFO` | 关键流程节点。**禁止循环体内打印** |
| `DEBUG` | 调试信息，生产默认关闭 |
### 要求
- 关键操作必须记日志（状态变更、支付等）
- 外部调用必须记日志（RPC、MQ）
- 禁止记录敏感信息（密码、身份证、手机号）
---
## 8. Entity & Enum
### Entity 必须注解
```java
@Data
@Accessors(chain = true)
public class BaseSkuCardContent {
    private String skuId;
    private ImageStruct afterMediaImage;
}
```
- 继承时加 `@EqualsAndHashCode(callSuper = true)`
- 需要 Builder 时加 `@Builder`
### Enum 标准写法
- 使用 `private static final Map` 静态映射表加速查询
- 提供 `getEnumByName(String)` 便捷查询方法
---
## 9. 配置管理
> 详见：[apollo-client-overview.md](apollo-client-overview.md)
- 所有动态开关/阈值使用 Apollo，**禁止硬编码**
- 简单值用 `@Value("${key:default}")`，复杂对象用 `@ApolloJsonValue`
- Key 命名：`<业务域>.<模块>.<功能>.<属性>`（如 `mall.shop.enable_new_ui`）
- 配置变更必须向前兼容
---
## 10. Redis 缓存
> 详见：[third-party-libraries.md](thirdparty-lib-guideline.md) 缓存中间件章节
- Key 格式：`业务:模块:功能:参数`，定义在 `RedisKeyConstant` 常量类中
- 批量操作使用 **Pipeline**
- 热点数据必须缓存，设置合理过期时间
- ZSet 使用 `zremrangeByRank` 限制大小
---
## 11. 设计模式
当有多个相似业务流程时，优先选用：
| 模式 | 场景 | 示例 |
|------|------|------|
| **模板方法** | 多种类型的 Feed 处理流程 | `AbstractDiffTypeFeed` → `processDiffTypeFeed()` |
| **工厂** | 根据类型返回不同策略 | `MediaImageProcessFactory.getDiffImageProcessStrategy()` |
| **建造者** | 复杂对象构建 | `@Builder` 注解 |
---
## 12. 代码质量检查清单
生成代码前，确保满足：
**结构**
- [ ] 类在正确的包结构下，命名符合规范
- [ ] 使用了合适的设计模式
**调用**
- [ ] RPC 使用 `RpcCallUtil` 封装，设置了超时
- [ ] 批量操作 `Lists.partition()` 分批
- [ ] 异步使用 `FutureUtil` / `GroupExecuteUtil`，禁止 `new Thread()`
**健壮性**
- [ ] 异常处理按场景分类，未过度 try-catch
- [ ] 监控代码 try-catch 保护
- [ ] 核心链路有降级策略
**规范**
- [ ] Entity 有 `@Data` + `@Accessors(chain = true)`
- [ ] 日志格式 `[ClassName.methodName]`，禁止循环内 INFO
- [ ] 配置用 Apollo，禁止硬编码
- [ ] Redis Key 用常量，批量用 Pipeline
- [ ] ThreadLocal 用 `try-finally` 清理
