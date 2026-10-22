---
name: 分布式锁使用指南
stages:
  - coding
source: base
priority: high
---
# 分布式锁使用指南
## 结论：不引入 Redisson
monorepo 内无任何服务使用 Redisson。业务场景为用户粒度防重锁，不需要可重入、RedLock 等高级特性，引入只增复杂度。
---
## 实现对比
| 实现 | 方法 | 释放方式 | 安全性 | 推荐度 |
|------|------|----------|--------|--------|
| `DistributeLockUtil` V2 | `lockAndUpdateV2` / `lockAndExecuteV2` | Lua `eval` 原子 compare-and-delete + acquired 保护 + 重试间隔 | ✅ 安全 | **新增场景必须用** |
| `DistributeLockUtil` V1 | `lockAndUpdate` / `lockAndExecute` | `evalsha` 硬编码 SHA（NOSCRIPT 风险）+ busy-wait | ⚠️ 有风险 | @Deprecated，存量勿新增 |
| `RedisLock` | `lockWithFunc` | get + del（非原子） | ⚠️ 极小概率误删 | 历史裂变活动存量，不扩展 |
| `SimpleJedisLock` | `getLock` / `releaseLock` | 直接 DEL，无 token 校验 | ❌ 有误删风险 | 存量勿新增，逐步迁移 |
Lock Key 常量统一定义在 `common/constants/LockKeyFormat.java`，格式：`业务_动作:{参数}`。
---
## 推荐用法（DistributeLockUtil V2）
```java
// lockAndExecuteV2：加锁 → 执行 → 原子释放，返回泛型结果；获锁失败返回 null
Result result = DistributeLockUtil.lockAndExecuteV2(jedis, lockKey, ttlInMs, maxTryCount, () -> {
    // 业务逻辑
    return result;
});
// lockAndUpdateV2：加锁 → 执行 → 原子释放，返回 Boolean；获锁失败/执行失败返回 false
boolean ok = DistributeLockUtil.lockAndUpdateV2(jedis, lockKey, ttlInMs, maxTryCount, () -> {
    // 业务逻辑
    return true;
});
```
**TTL 设置原则**：根据业务最长耗时留足余量。多数场景 5000ms，MQ 消费幂等场景 1000ms。
---
## DistributeLockUtil 设计要点
### 加锁
`SET key requestId NX PX ttl`，原子性，key 不存在时才写入，写入值为 UUID requestId。
### 解锁（Lua eval，原子 compare-and-delete）
```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```
只有持锁者（requestId 匹配）才能删除，防止误删他人锁。
用 `eval` 直接传脚本而非 `evalsha`，避免 Redis 重启后脚本缓存清除导致 NOSCRIPT 异常、锁无法释放。
### 重试
获锁失败时每次间隔 50ms 再试，避免 busy-wait 把 maxTryCount 次在微秒内打完。仅在非最后一次失败时 sleep。
### finally 保护
用 `acquired` 标记是否加锁成功，未加锁时跳过解锁操作，避免无意义的 Redis 请求。
---
## 常见错误场景（来源：内部规范文档）
### 1. 锁误删除
**场景**：线程 A 持锁，执行超时，TTL 到期后线程 B 获锁；A 执行完毕 DEL，删掉了 B 的锁。  
**解法**：Lua compare-and-delete，只有 requestId 匹配才删。`DistributeLockUtil` 已实现。
### 2. 超时导致并发执行
**场景**：线程 A 持锁时间超过 TTL，锁自动过期，B 获锁，A、B 并发执行临界区。  
**解法**：TTL 设置足够长（覆盖业务最长路径）。watch dog（Redisson 特性）此项目不引入。
### 3. 不可重入
同一线程持锁后再次请求会失败。`DistributeLockUtil` 是不可重入锁，业务层避免递归调用加锁方法。
