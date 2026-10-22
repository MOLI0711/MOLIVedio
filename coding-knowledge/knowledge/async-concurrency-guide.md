---
name: 异步并发处理指南
stages:
  - coding
source: base
priority: medium
---
# 异步并发处理指南
> 本文档基于后端项目的异步并发最佳实践提炼，包含使用场景判断、工具选型和代码模板。
---
## 使用场景判断
**使用异步并发：**
- 多个 RPC 调用可以并行执行
- 批量数据处理可以分片并行
- IO 密集型操作（数据库、Redis、HTTP）
**不使用异步并发：**
- 单个 RPC 调用且耗时 < 50ms
- CPU 密集型计算
- 后续逻辑强依赖前一步结果
---
## 工具选型
| 工具 | 使用场景 | 超时控制 | 异常处理 |
|------|---------|---------|---------|
| **GroupExecuteUtil** | 批量元素并行处理（同类操作） | 统一超时 | 返回空列表 |
| **FutureUtil** | 多个不同异步任务并行 | 独立超时 | 返回默认值 |
| **手动 Lists.partition + Future** | 批量分批并发调用 RPC | 手动管理 | 手动处理 |
### 超时设置规范
| 依赖类型 | 超时时间 | 说明 |
|---------|---------|------|
| 强依赖 | 1500-2000ms | 核心数据，失败影响业务 |
| 弱依赖 | 1000ms | 辅助数据，失败可降级 |
---
## 模式一：多个异步任务并行（FutureUtil）
适用于多个不同的 RPC/IO 调用可以并行执行的场景。
```java
public DataList<LiveRoomCardV2Entity> doProcess(ProcContext context, DataList<LiveRoomCardV2Form> inputList) {
    
    // 1. 并行提交多个异步任务
    Future<Map<Long, TLiveRoomDetail>> roomInfoFuture = 
        FutureUtil.invokeTask(businessTaskExecutor, 
            () -> liveClient.getRoomInfoByIds(cardIdList, context.getUserId()));
    
    Future<Map<Long, Long>> roomIdFuture = 
        FutureUtil.invokeTask(businessTaskExecutor, 
            () -> liveClient.batchGetRoomByCardIds(cardIdList));
    
    Future<Map<String, List<ImageStream>>> imageMapFuture = 
        FutureUtil.invokeTask(businessTaskExecutor, 
            () -> getMediaControlImageMap(inputList.getDataList()));
    
    // 2. 等待所有任务完成，每个任务独立超时
    Map<Long, TLiveRoomDetail> roomInfoMap = 
        FutureUtil.getWithDefault(roomInfoFuture, 1500L, Maps.newHashMap());
    
    Map<Long, Long> roomIdMap = 
        FutureUtil.getWithDefault(roomIdFuture, 1000L, Maps.newHashMap());
    
    Map<String, List<ImageStream>> imageMap = 
        FutureUtil.getWithDefault(imageMapFuture, 1000L, Maps.newHashMap());
    
    // 3. 使用结果构建响应
    // ...
}
```
**要点**：
- 先提交所有任务，再统一等待结果
- 每个 Future 可设置不同超时时间（强依赖 vs 弱依赖）
- `getWithDefault` 超时或异常时返回默认值，不抛异常
---
## 模式二：批量分批并发（Lists.partition + Future）
适用于大量 ID 需要分批调用 RPC 查询的场景。
```java
private List<GoodsEntity> queryGoodsEntity(List<String> skuIds) {
    try {
        if (CollectionUtils.isEmpty(skuIds)) {
            return new ArrayList<>();
        }
        
        // 1. 分批：每批 100 个
        List<List<String>> batches = Lists.partition(skuIds, 100);
        log.info("queryGoodsEntity: partitioned into {} batches", batches.size());
        
        // 2. 为每个批次创建并发任务
        List<Future<List<GoodsEntity>>> futures = new ArrayList<>();
        for (List<String> batch : batches) {
            Future<List<GoodsEntity>> future = businessTaskExecutor.submit(
                () -> queryGoodsEntityBatch(batch)
            );
            futures.add(future);
        }
        
        // 3. 等待所有任务完成并合并结果
        List<GoodsEntity> allResults = new ArrayList<>();
        for (Future<List<GoodsEntity>> future : futures) {
            List<GoodsEntity> batchResult = FutureTaskHandler.getWithDefault(
                future, 2000L, new ArrayList<>()
            );
            if (CollectionUtils.isNotEmpty(batchResult)) {
                allResults.addAll(batchResult);
            }
        }
        
        log.info("queryGoodsEntity: success, returned {} items", allResults.size());
        return allResults;
        
    } catch (Exception e) {
        log.warn("queryGoodsEntity: exception occurred", e);
        return new ArrayList<>();
    }
}
```
**要点**：
- **批次大小**：一般 50-200，根据单次 RPC 耗时和数据量调整
- **异常隔离**：单个批次失败（超时/异常）不影响其他批次
- 在 Client 层封装分批逻辑时，可结合 `RpcCallUtil`：
```java
@Component
@Slf4j
public class ProductClient {
    
    private static final int BATCH_SIZE = 100;
    
    public Map<String, ProductInfo> batchGetProductInfo(List<String> productIds) {
        if (CollectionUtils.isEmpty(productIds)) {
            return Maps.newHashMap();
        }
        
        List<List<String>> batches = Lists.partition(productIds, BATCH_SIZE);
        List<Future<Map<String, ProductInfo>>> futures = new ArrayList<>();
        
        for (List<String> batch : batches) {
            futures.add(businessTaskExecutor.submit(() -> {
                BatchGetProductInfoRequest request = new BatchGetProductInfoRequest();
                request.setProductIds(batch);
                return RpcCallUtil.execute(
                    "ProductClient", "productService.batchGetProductInfo",
                    () -> productService.batchGetProductInfo(ContextHelper.getContext(), request),
                    null, false, true, Maps.newHashMap(), true, request
                ).getProductInfoMap();
            }));
        }
        
        Map<String, ProductInfo> allResults = new HashMap<>();
        for (Future<Map<String, ProductInfo>> future : futures) {
            allResults.putAll(FutureTaskHandler.getWithDefault(future, 2000L, Maps.newHashMap()));
        }
        return allResults;
    }
}
```
---
## 模式三：批量并行执行（GroupExecuteUtil）
适用于对一组元素执行相同操作并统一超时的场景。
```java
public BatchGetResourcePlanResponse batchGetResourcePlan(Context context, BatchGetResourcePlanRequest request) {
    try {
        List<ProcContext> procContextList = batchRecallResource(context, request);
        
        // 使用 GroupExecuteUtil 并行执行
        List<ResourcePlanInfo> resourcePlanInfoList = GroupExecuteUtil
            .execute("BATCH_GET_RESOURCE_PLAN", procContextList,
                procContext -> getSingleResourcePlan(procContext),  // 每个元素执行的逻辑
                engineTaskExecutor,                                 // 线程池
                2000L, TimeUnit.MILLISECONDS)                       // 统一超时
            .stream()
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
        
        return buildResponse(resourcePlanInfoList);
        
    } catch (Throwable throwable) {
        log.error("batchGetResourcePlan fail, global degrade", throwable);
        Cat.logEvent("DEGRADE_STRATEGY", "GLOBAL_EXCEPTION_DEGRADE_CACHE");
        return getResponseDegradeInfo(request);
    }
}
```
---
## 并发基本规范
1. **禁止** `new Thread()`，使用项目统一线程池：
   - `businessTaskExecutor` - 通用业务任务
   - `engineTaskExecutor` - 引擎/召回任务
   - `traceExecutor` - 链路追踪任务
2. **ThreadLocal 清理**：必须使用 `try-finally` 块：
   ```java
   try {
       threadLocal.set(value);
       // ...
   } finally {
       threadLocal.remove();
   }
   ```
3. **所有 Future 必须设置超时**，禁止无限等待。
