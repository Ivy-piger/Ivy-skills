---
name: java-coding-standards
description: "Spring Boot 服务的 Java 编码规范：命名、可变性、Optional 使用、Stream、异常处理、泛型及项目结构布局。"
origin: ECC
---

# Java 编码规范

Spring Boot 服务中可读、可维护 Java（17+）代码的编码标准。

## 何时启用

- 在 Spring Boot 项目中编写或审查 Java 代码
- 强制执行命名、可变性或异常处理规范
- 使用 record、sealed class 或模式匹配（Java 17+）
- 审查 Optional、Stream 或泛型的使用
- 组织包结构和项目布局

## 核心原则

- **清晰优于巧妙**（Prefer clarity over cleverness）
- **默认不可变**；尽量减少共享可变状态
- **快速失败**，并抛出有意义的异常
- 保持一致的命名和包结构

## 命名规范

```java
// ✅ 通过：类 / Record：PascalCase（帕斯卡命名法）
public class MarketService {}
public record Money(BigDecimal amount, Currency currency) {}

// ✅ 通过：方法 / 字段：camelCase（驼峰命名法）
private final MarketRepository marketRepository;
public Market findBySlug(String slug) {}

// ✅ 通过：常量：UPPER_SNAKE_CASE（全大写下划线分隔）
private static final int MAX_PAGE_SIZE = 100;
```

## 不可变性

```java
// ✅ 推荐：优先使用 record 和 final 字段
public record MarketDto(Long id, String name, MarketStatus status) {}

public class Market {
  private final Long id;
  private final String name;
  // 只提供 getter，不提供 setter
}
```

## Optional 使用规范

```java
// ✅ 通过：find* 方法返回 Optional
Optional<Market> market = marketRepository.findBySlug(slug);

// ✅ 通过：使用 map / flatMap 而不是 get()
return market
    .map(MarketResponse::from)
    .orElseThrow(() -> new EntityNotFoundException("Market not found"));
```

## Stream 最佳实践

```java
// ✅ 通过：使用 Stream 进行转换，保持流水线简短
List<String> names = markets.stream()
    .map(Market::name)
    .filter(Objects::nonNull)
    .toList();

// ❌ 避免：复杂的嵌套 Stream；优先使用普通循环以提高可读性
```

## 异常处理

- 领域错误使用非受检异常（unchecked exception）；用上下文包装技术异常
- 创建领域特定的异常类（如 `MarketNotFoundException`）
- 避免宽泛的 `catch (Exception ex)`，除非是重新抛出或统一日志记录

```java
throw new MarketNotFoundException(slug);
```

## 泛型与类型安全

- 避免原始类型（raw type）；声明泛型参数
- 可复用的工具类优先使用有界泛型（bounded generics）

```java
public <T extends Identifiable> Map<Long, T> indexById(Collection<T> items) { ... }
```

## 项目结构（Maven / Gradle）

```
src/main/java/com/example/app/
  config/       （配置层）
  controller/   （控制器层）
  service/      （服务层）
  repository/   （仓储层）
  domain/       （领域模型）
  dto/          （数据传输对象）
  util/         （工具类）
src/main/resources/
  application.yml
src/test/java/...（与 main 结构镜像对应）
```

## 格式化与风格

- 统一使用 2 或 4 空格缩进（遵循项目规范）
- 每个文件只包含一个公共顶层类型
- 方法保持简短且专注；抽取辅助方法
- 成员顺序：常量 → 字段 → 构造器 → 公共方法 → protected → private

## 需要避免的代码坏味

| 坏味 | 改进方式 |
|------|---------|
| 过长的参数列表 | 使用 DTO / Builder 模式 |
| 深层嵌套 | 使用提前返回（early return） |
| 魔法数字 | 定义命名常量 |
| 静态可变状态 | 优先使用依赖注入 |
| 静默捕获块 | 记录日志并处理，或重新抛出 |

## 日志规范

```java
private static final Logger log = LoggerFactory.getLogger(MarketService.class);
log.info("fetch_market slug={}", slug);
log.error("failed_fetch_market slug={}", slug, ex);
```

## 空值处理

- 仅在不可避免时接受 `@Nullable`；否则使用 `@NonNull`
- 输入参数使用 Bean Validation 注解（`@NotNull`、`@NotBlank`）

## 测试期望

- 使用 **JUnit 5 + AssertJ** 实现流式断言
- 使用 **Mockito** 进行模拟；尽量避免部分模拟（partial mocks）
- 优先使用确定性测试；不要有隐藏的 sleep 等待

---

> **记住**：保持代码意图明确、类型安全、可观测。除非经过验证，否则不要为了微优化而牺牲可维护性。
