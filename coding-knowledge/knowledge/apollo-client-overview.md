# Apollo 配置中心使用指南
> **版本**: Apollo Client 1.5.1  
> **用途**: 分布式配置中心，用于管理动态配置和开关
---
## 1. 配置注入方式
| 注解 | 用途 | 示例 |
|------|------|------|
| `@Value` | 注入简单类型配置 | `@Value("${key:default}")` |
| `@ApolloJsonValue` | 注入 JSON 反序列化为对象 | `@ApolloJsonValue("${key:{}}")` |
| `@ApolloConfig` | 注入 Config 对象（用于动态读取） | `@ApolloConfig("namespace")` |
### 使用示例
```java
@Component
public class MyConfig {
    
    // 简单配置（带默认值）
    @Value("${mall.shop.enable_new_ui:false}")
    private Boolean enableNewUi;
    
    // JSON 对象
    @ApolloJsonValue("${goods.config:{}}")
    private GoodsConfig goodsConfig;
    
    // JSON 数组
    @ApolloJsonValue("${exchange.config:[]}")
    private List<ExchangeConfig> exchangeConfigs;
    
    // Config 对象（用于动态读取 key）
    @ApolloConfig("goods-config")
    private Config config;
    
    public String getDynamicValue(String key) {
        return config.getProperty(key, "");
    }
}
```
---
## 2. 启动配置
### @EnableApolloConfig
```java
@SpringBootApplication
@EnableApolloConfig({"application", "exchange.config", "common-config"})
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
### application.yml
```yaml
apollo:
  bootstrap:
    enabled: true
    namespaces: application, goods-config, common-config
```
### app.properties
```properties
app.id=demo-app
apollo.autoUpdateInjectedSpringProperties=true
```
---
## 3. 配置变更监听
```java
@Component
public class ConfigChangeListener {
    
    @ApolloConfig("application")
    private Config config;
    
    @PostConstruct
    public void init() {
        config.addChangeListener(changeEvent -> {
            for (String key : changeEvent.changedKeys()) {
                ConfigChange change = changeEvent.getChange(key);
                log.info("Config changed - key: {}, old: {}, new: {}, type: {}",
                    key, change.getOldValue(), change.getNewValue(), change.getChangeType());
            }
        });
    }
}
```
> **注意**：1.5.1 版本不支持 `@ApolloConfigChangeListener` 注解，需要使用编程方式。
---
## 4. 命名规范
| 类型 | 规范 | 示例 |
|------|------|------|
| Namespace | 小写，横线分隔 | `goods-config` |
| Key | 小写，点分隔，带模块前缀 | `mall.shop.enable_new_ui` |
| Key 格式 | `<业务域>.<模块>.<功能>.<属性>` | `activity.channel.pageSize` |
---
## 5. 最佳实践
- 所有动态开关/阈值使用 Apollo，**禁止硬编码**
- 配置变更**必须向前兼容**
- Key 必须带模块前缀，避免冲突
- `@ApolloJsonValue` 必须提供默认值（如 `{}` 或 `[]`）
---
## 6. 常见问题
| 问题 | 排查步骤 |
|------|---------|
| 配置不生效 | 1. 检查 `app.id` 2. 检查 `apollo.bootstrap.enabled=true` 3. 检查 namespace 是否正确 4. 查看本地缓存文件 |
| `@ApolloJsonValue` 如何指定 namespace？ | 1.5.1 不支持直接指定，需在 `@EnableApolloConfig` 或 `application.yml` 中声明 |
| 配置热更新 | 默认支持（5 分钟轮询），也可通过 `addChangeListener` 实时监听 |
---
## 7. 1.5.1 版本限制
| 特性 | 支持情况 |
|------|---------|
| `@ApolloJsonValue` | ✅ |
| `@ApolloConfig` | ✅ |
| `@ApolloConfigChangeListener` | ❌ 需编程方式 |
| namespace 参数 | ❌ 需全局声明 |
---
**相关文档**：
- [code-standards.md](code-standards.md) - 配置管理规则
- [third-party-libraries.md](thirdparty-lib-guideline.md) - Apollo 使用索引
