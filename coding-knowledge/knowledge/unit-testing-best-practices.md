# 单元测试最佳实践指南
---
## 1. 测试框架概览
### 1.1 技术栈
| 组件 | 版本 | 用途 |
|------|------|------|
| **JUnit 5 (Jupiter)** | 5.x | 主测试框架 ⚠️ **注意：使用 JUnit 5，不是 JUnit 4** |
| **Mockito** | MockitoExtension | Mock 依赖注入 |
| **Spring Boot Test** | 2.1.6.RELEASE | 集成测试 |
| **JaCoCo** | 0.8.4 | 代码覆盖率 |
| **ReflectionTestUtils** | Spring | 私有方法测试 |
> 🔔 **重要提醒**：本项目使用 **JUnit 5 (Jupiter)**，导包时请注意：
> - ✅ 正确: `import org.junit.jupiter.api.Test;`
> - ❌ 错误: `import org.junit.Test;` (这是 JUnit 4)
### 1.2 JUnit 5 核心注解
| 注解 | 用途 | 示例 |
|------|------|------|
| `@Test` | 标记测试方法 | `@Test void testMethod() {}` |
| `@BeforeEach` | 每个测试方法前执行 | 初始化测试数据 |
| `@AfterEach` | 每个测试方法后执行 | 清理资源 |
| `@BeforeAll` | 所有测试前执行一次（需要 static） | 全局初始化 |
| `@AfterAll` | 所有测试后执行一次（需要 static） | 全局清理 |
| `@DisplayName` | 自定义测试显示名称 | `@DisplayName("用户登录测试")` |
| `@Disabled` | 禁用测试 | 暂时跳过某些测试 |
| `@ParameterizedTest` | 参数化测试 | 多组数据测试同一逻辑 |
### 1.3 JUnit 5 常用断言方法
```java
import static org.junit.jupiter.api.Assertions.*;
// 相等性断言
assertEquals(expected, actual);
assertEquals(expected, actual, "自定义错误消息");
assertNotEquals(unexpected, actual);
// 布尔断言
assertTrue(condition);
assertTrue(condition, "条件应为 true");
assertFalse(condition);
// Null 检查
assertNull(object);
assertNotNull(object);
// 同一性检查（判断是否为同一对象）
assertSame(expected, actual);
assertNotSame(unexpected, actual);
// 数组/集合断言
assertArrayEquals(expectedArray, actualArray);
assertIterableEquals(expectedList, actualList);
// 异常断言
assertThrows(IllegalArgumentException.class, () -> {
    service.methodThatThrowsException();
});
// 超时断言
assertTimeout(Duration.ofSeconds(1), () -> {
    service.fastMethod();
});
// 组合断言（全部执行，即使有失败）
assertAll(
    () -> assertEquals(expected1, actual1),
    () -> assertEquals(expected2, actual2),
    () -> assertTrue(condition)
);
```
### 1.4 JUnit 4 vs JUnit 5 对比
| 功能 | JUnit 4 | JUnit 5 (Jupiter) |
|------|---------|-------------------|
| **包路径** | `org.junit.*` | `org.junit.jupiter.api.*` |
| **测试注解** | `@Test` | `@Test` |
| **前置方法** | `@Before` | `@BeforeEach` |
| **后置方法** | `@After` | `@AfterEach` |
| **类前置** | `@BeforeClass` (static) | `@BeforeAll` (static) |
| **类后置** | `@AfterClass` (static) | `@AfterAll` (static) |
| **忽略测试** | `@Ignore` | `@Disabled` |
| **断言类** | `org.junit.Assert` | `org.junit.jupiter.api.Assertions` |
| **期望异常** | `@Test(expected=...)` | `assertThrows(...)` |
| **超时** | `@Test(timeout=...)` | `assertTimeout(...)` |
| **测试类修饰符** | 必须 `public` | 可以是 package-private |
| **测试方法修饰符** | 必须 `public` | 可以是 package-private |
### 1.5 覆盖率要求
| 模块类型 | 分支覆盖率要求 |
|---------|---------------|
| 核心工具类（Util） | ≥ **80%** |
| 核心引擎逻辑（Engine） | ≥ **80%** |
| 业务 Solution | ≥ **60%** |
| Client 封装 | ≥ **50%** |
---
## 2. AIR 原则（必须遵守）
| 原则 | 英文 | 说明 |
|------|------|------|
| **A** | Automatic | 自动化执行，无需人工干预 |
| **I** | Independent | 测试之间相互独立，无执行顺序依赖 |
| **R** | Repeatable | 可重复执行，结果一致 |
---
## 3. Mock 原则
### 3.1 基本规则
- 单元测试 **不应依赖** 真实的外部数据库或 RPC 服务
- **必须使用** Mockito 等框架模拟外部依赖
- Mock 对象应该尽量模拟真实行为
### 3.2 Mock 适用场景
| 场景 | 是否 Mock | 说明 |
|------|----------|------|
| 外部 RPC 服务 | ✅ Mock | 避免网络依赖 |
| 数据库访问 | ✅ Mock | 避免数据依赖 |
| Redis 缓存 | ✅ Mock | 避免环境依赖 |
| 内部工具方法 | ❌ 不 Mock | 保持真实逻辑 |
| 静态工具类 | 视情况 | 必要时使用 MockedStatic |
---
## 4. 测试类模板
### 4.1 单元测试标准模板
```java
package com.example.project.xxx;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;
/**
 * XxxService 单元测试
 * 
 * @author 作者名
 */
@ExtendWith(MockitoExtension.class)
class XxxServiceTest {
    @Mock
    private DependencyA dependencyA;   // Mock 外部依赖
    
    @Mock
    private DependencyB dependencyB;   // Mock 外部依赖
    
    @InjectMocks
    private XxxService xxxService;     // 被测对象
    
    @BeforeEach
    void setUp() {
        // 初始化测试数据（如有需要）
    }
    
    @Test
    void testMethod_正常场景_返回预期结果() {
        // Arrange - 准备测试数据
        String input = "test";
        String expected = "result";
        when(dependencyA.method(input)).thenReturn(expected);
        
        // Act - 执行被测方法
        String actual = xxxService.method(input);
        
        // Assert - 验证结果
        assertEquals(expected, actual);
        verify(dependencyA).method(input);
    }
    
    @Test
    void testMethod_异常场景_抛出异常() {
        // Arrange
        when(dependencyA.method(anyString()))
            .thenThrow(new RuntimeException("模拟异常"));
        
        // Act & Assert
        assertThrows(RuntimeException.class, 
            () -> xxxService.method("test"));
    }
}
```
### 4.2 集成测试模板
```java
package com.example.project.xxx;
import com.example.project.BaseTest;
import org.junit.Test;
import org.springframework.beans.factory.annotation.Autowired;
/**
 * XxxService 集成测试
 */
public class XxxServiceIntegrationTest extends BaseTest {
    @Autowired
    private XxxService xxxService;
    
    @Test
    public void testIntegration() {
        // 集成测试逻辑
    }
}
```
---
## 5. 命名规范
### 5.1 测试类命名
| 类型 | 命名规范 | 示例 |
|------|---------|------|
| 单元测试类 | `{被测类}Test` | `MonitorAspectTest` |
| 集成测试类 | `{被测类}IntegrationTest` | `XxxServiceIntegrationTest` |
### 5.2 测试方法命名
**格式**: `test{方法名}_{场景描述}_{预期结果}`
| 示例 | 说明 |
|------|------|
| `testAroundMonitor_Success` | 正常场景成功 |
| `testAroundMonitor_WithException` | 异常场景 |
| `testExtractCount_DataList` | 测试 DataList 类型 |
| `testExtractCount_Null` | 测试 null 输入 |
| `testFindByName_NotFound` | 测试未找到情况 |
---
## 6. AAA 模式
每个测试方法必须遵循 **Arrange-Act-Assert** 模式：
```java
@Test
void testMethod_场景_预期() {
    // ============ Arrange（准备） ============
    // 1. 准备测试数据
    String input = "test";
    String expected = "result";
    
    // 2. 设置 Mock 行为
    when(dependency.method(input)).thenReturn(expected);
    
    // ============ Act（执行） ============
    String actual = service.method(input);
    
    // ============ Assert（断言） ============
    // 1. 验证返回值
    assertEquals(expected, actual);
    
    // 2. 验证 Mock 调用
    verify(dependency).method(input);
    verify(dependency, times(1)).method(anyString());
}
```
---
## 7. 常用测试技巧
### 7.1 静态方法 Mock
```java
@Test
void testWithStaticMethod() {
    try (MockedStatic<Cat> catMock = mockStatic(Cat.class)) {
        // Arrange
        catMock.when(() -> Cat.newTransaction(anyString(), anyString()))
               .thenReturn(transaction);
        
        // Act
        Object result = service.method();
        
        // Assert
        assertNotNull(result);
    }
}
```
### 7.2 私有方法测试
```java
@Test
void testPrivateMethod() {
    // 使用 ReflectionTestUtils 调用私有方法
    Object result = ReflectionTestUtils.invokeMethod(
        service, 
        "privateMethodName", 
        arg1, arg2
    );
    
    assertEquals(expected, result);
}
```
### 7.3 私有字段设置
```java
@Test
void testWithPrivateField() {
    // 设置私有字段值
    ReflectionTestUtils.setField(service, "fieldName", mockValue);
    
    // 执行测试
    Object result = service.method();
    
    // 验证
    assertNotNull(result);
}
```
### 7.4 测试辅助类
```java
@ExtendWith(MockitoExtension.class)
class XxxTest {
    
    // ============ 辅助类定义 ============
    
    public static class TestService {
        public void testMethod(DataList<String> dataList) {}
        public void recallMethod() {}
        public void fillMethod() {}
    }
    
    public static class TestObject {
        private DataList<String> dataList;
        private String name;
        
        // getter/setter
    }
    
    // ============ 工厂方法 ============
    
    private Monitor createMockMonitor() {
        Monitor annotation = mock(Monitor.class);
        when(annotation.category()).thenReturn("FILL");
        when(annotation.name()).thenReturn("test_method");
        return annotation;
    }
    
    private DataList<String> createDataList(List<String> items) {
        DataList<String> dataList = new DataList<>();
        dataList.setDataList(items);
        return dataList;
    }
}
```
### 7.5 参数化测试（JUnit 5）
JUnit 5 提供了强大的参数化测试功能，可以用同一测试方法测试多组数据。
#### 7.5.1 基础参数化测试
```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
// 字符串数组参数
@ParameterizedTest
@ValueSource(strings = {"value1", "value2", "value3"})
void testWithStringValues(String input) {
    assertNotNull(service.process(input));
}
// 整数参数
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 5, 10})
void testWithIntValues(int count) {
    assertTrue(count > 0);
}
```
#### 7.5.2 CSV 参数化测试
```java
// CSV 格式参数（逗号分隔）
@ParameterizedTest
@CsvSource({
    "input1, expected1",
    "input2, expected2",
    "input3, expected3"
})
void testWithCsvInputs(String input, String expected) {
    String actual = service.method(input);
    assertEquals(expected, actual);
}
// 带自定义显示名称
@ParameterizedTest(name = "{index} - 测试输入 {0} 应返回 {1}")
@CsvSource({
    "apple,  5",
    "banana, 6",
    "cherry, 6"
})
void testStringLength(String input, int expectedLength) {
    assertEquals(expectedLength, input.length());
}
```
#### 7.5.3 方法参数源
```java
@ParameterizedTest
@MethodSource("provideTestData")
void testWithMethodSource(String input, int expected) {
    assertEquals(expected, service.calculate(input));
}
// 参数提供方法（必须是 static）
static Stream<Arguments> provideTestData() {
    return Stream.of(
        Arguments.of("test1", 1),
        Arguments.of("test2", 2),
        Arguments.of("test3", 3)
    );
}
```
#### 7.5.4 Null 和空值测试
```java
@ParameterizedTest
@NullSource
void testWithNull(String input) {
    assertThrows(NullPointerException.class,
        () -> service.process(input));
}
@ParameterizedTest
@EmptySource
void testWithEmptyString(String input) {
    assertTrue(input.isEmpty());
}
// 组合 Null 和空值
@ParameterizedTest
@NullAndEmptySource
void testWithNullAndEmpty(String input) {
    assertTrue(input == null || input.isEmpty());
}
```
---
## 8. 断言最佳实践
### 8.1 常用断言
```java
// 相等断言
assertEquals(expected, actual);
assertEquals(expected, actual, "自定义错误消息");
// null 断言
assertNull(object);
assertNotNull(object);
// 布尔断言
assertTrue(condition);
assertFalse(condition);
// 异常断言
assertThrows(RuntimeException.class, () -> service.method());
// 集合断言
assertEquals(3, list.size());
assertTrue(list.isEmpty());
assertTrue(list.contains(element));
```
### 8.2 验证 Mock 调用
```java
// 验证调用次数
verify(dependency).method(arg);           // 调用1次
verify(dependency, times(2)).method(arg); // 调用2次
verify(dependency, never()).method(arg);  // 从未调用
verify(dependency, atLeastOnce()).method(arg); // 至少1次
// 验证调用顺序
InOrder inOrder = inOrder(dep1, dep2);
inOrder.verify(dep1).method1();
inOrder.verify(dep2).method2();
// 验证无更多交互
verifyNoMoreInteractions(dependency);
```
---
## 9. 测试目录结构
```
src/test/java/com/example/project/
├── BaseTest.java                           # 集成测试基类
├── client/                                 # 客户端测试
│   ├── CacheTest.java
│   └── GrayUtilTest.java
├── engine/                                 # 引擎测试
│   ├── ResourceEngineServiceTest.java
│   ├── common/util/
│   │   └── ListUtilTest.java
│   └── util/
│       ├── ConfigMappingConvertTest.java
│       └── JsonCompareUtilTest.java
└── solutions/                              # 业务测试
    ├── RecommendSolutionTest.java
    ├── OrderSolutionTest.java
    └── common/
        ├── aspect/
        │   └── MonitorAspectTest.java
        ├── biz/
        └── util/
```
---
## 10. 测试检查清单
### 代码提交前必须完成：
- [ ] 单元测试覆盖核心逻辑
- [ ] 核心模块覆盖率达到 80%
- [ ] 使用 Mockito 模拟外部依赖
- [ ] 遵循 AAA 模式（Arrange-Act-Assert）
- [ ] 测试命名清晰表达意图
- [ ] 测试之间相互独立
- [ ] 运行 `gradle test` 全部通过
- [ ] 无硬编码的测试数据路径
### 测试命令
```bash
# 运行所有单元测试
gradle test
# 运行特定测试类
gradle test --tests "MonitorAspectTest"
# 运行特定测试方法
gradle test --tests "MonitorAspectTest.testAroundMonitor_Success"
# 生成覆盖率报告
gradle jacocoTestReport
```
---
## 11. 参考示例
项目中的标杆测试文件：
- [`MonitorAspectTest.java`](../../src/test/java/com/example/project/solutions/common/aspect/MonitorAspectTest.java) - 完整的单元测试示例
- [`BaseTest.java`](../../src/test/java/com/example/project/BaseTest.java) - 集成测试基类
---
---
## 12. 单测故障排查清单
当遇到"单测跑不起来"的问题时，按以下优先级逐一检查。详见 Skill: `unit-test-troubleshooting`。
### 12.1 目录与命名规范（最常见，优先级最高）
**检查清单：**
- [ ] 测试类是否在 `src/test/java` 目录下？
- [ ] 测试类名是否以 `Test` 或 `Tests` 结尾？（例：`UserServiceTest.java`）
- [ ] 测试方法是否是 `public` 且标注 `@Test`？
- [ ] 包结构是否与 `src/main/java` 保持一致？
**常见错误：**
- ❌ 测试类在 `src/main/java` 中
- ❌ 测试类名为 `UserServiceTestCase.java`（不以 Test 结尾）
- ❌ 测试方法是 `private` 或 `protected`
### 12.2 Lombok 配置问题（高频坑）
**问题现象：**
- 编译错误：`cannot find symbol` 或 `@Data` 等注解无法识别
- 主代码编译失败，导致测试无法运行
**解决方案：**
Lombok 需要在主代码和测试代码中都配置：
```groovy
// 主代码
compileOnly 'org.projectlombok:lombok:1.18.22'
annotationProcessor 'org.projectlombok:lombok:1.18.22'
// 测试代码
testCompileOnly 'org.projectlombok:lombok:1.18.22'
testAnnotationProcessor 'org.projectlombok:lombok:1.18.22'
```
**IDE 配置（IntelliJ IDEA）：**
1. Settings → Build, Execution, Deployment → Compiler → Annotation Processors
2. 勾选 "Enable annotation processing"
3. 勾选 "Obtain processors from project classpath"
### 12.3 Spring 上下文加载失败
**问题现象：**
- 错误：`ApplicationContext` 加载失败
- 错误：`NoSuchBeanDefinitionException`
- 超时错误：测试启动时间过长
**解决方案：**
**原则：纯单元测试不加 `@SpringBootTest`**
```java
// ❌ 错误：启动整个 Spring 容器，速度慢
@SpringBootTest
class UserServiceTest {
    @Autowired
    private UserService userService;
}
// ✅ 正确：只测试 Service 逻辑，速度快
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
}
```
**创建测试配置文件：**
```yaml
# src/test/resources/application.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: create-drop
```
### 12.4 Gradle 版本兼容性
**问题现象：**
- 测试任务被跳过（SKIPPED）
- 错误：`Unsupported class-file format`
- 错误：`Module system` 相关错误
**解决方案：**
运行诊断命令：
```bash
# 查看 Gradle 版本
./gradlew --version
# 查看详细的测试执行信息
./gradlew clean test --info 2>&1 | grep -i "test\|skip\|fail"
# 强制重新编译和运行测试
./gradlew clean test --no-build-cache
```
**版本兼容性表：**
| Spring Boot | 推荐 Gradle | Java |
|-------------|-----------|------|
| 2.1.6 | 5.x, 6.x | 8, 11 |
| 2.7.x | 6.x, 7.x | 8, 11, 17 |
| 3.x | 7.x, 8.x | 17+ |
**Gradle 配置优化：**
```groovy
test {
    useJUnitPlatform() // 启用 JUnit 5
    
    // 强制每次运行测试，防止 Gradle 缓存导致跳过
    outputs.upToDateWhen { false }
    
    testLogging {
        events "passed", "skipped", "failed"
        showStandardStreams = true
    }
}
```
### 12.5 快速诊断命令
```bash
# 1. 查看 Gradle 和 Java 版本
./gradlew --version && java -version
# 2. 查看依赖树
./gradlew dependencies --configuration testCompileClasspath | grep -E "junit|mockito|lombok"
# 3. 清理并编译
./gradlew clean compileJava compileTestJava
# 4. 运行所有测试（详细输出）
./gradlew test --info
# 5. 运行特定测试类
./gradlew test --tests "UserServiceTest" -i
# 6. 查看测试报告
open build/reports/tests/test/index.html
```
---
