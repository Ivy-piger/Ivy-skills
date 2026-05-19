---
name: springboot-patterns
description: Spring Boot 架构模式、REST API 设计、分层服务、数据访问、缓存、异步处理及日志。用于 Java Spring Boot 后端开发。
origin: ECC
---

# Spring Boot 开发模式

适用于可扩展、生产级服务的 Spring Boot 架构和 API 模式。

## 何时启用

- 使用 Spring MVC 或 WebFlux 构建 REST API
- 构建 controller → service → repository 分层架构
- 配置 Spring Data JPA、缓存或异步处理
- 添加校验、异常处理或分页
- 设置 dev / staging / production 环境 profile
- 使用 Spring Events 或 Kafka 实现事件驱动模式

## REST API 结构

```java
@RestController
@RequestMapping("/api/markets")
@Validated
class MarketController {
  private final MarketService marketService;

  MarketController(MarketService marketService) {
    this.marketService = marketService;
  }

  @GetMapping
  ResponseEntity<Page<MarketResponse>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    Page<Market> markets = marketService.list(PageRequest.of(page, size));
    return ResponseEntity.ok(markets.map(MarketResponse::from));
  }

  @PostMapping
  ResponseEntity<MarketResponse> create(@Valid @RequestBody CreateMarketRequest request) {
    Market market = marketService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(MarketResponse.from(market));
  }
}
```

## 仓储模式（Spring Data JPA）

```java
public interface MarketRepository extends JpaRepository<MarketEntity, Long> {
  @Query("select m from MarketEntity m where m.status = :status order by m.volume desc")
  List<MarketEntity> findActive(@Param("status") MarketStatus status, Pageable pageable);
}
```

## 事务性服务层

```java
@Service
public class MarketService {
  private final MarketRepository repo;

  public MarketService(MarketRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public Market create(CreateMarketRequest request) {
    MarketEntity entity = MarketEntity.from(request);
    MarketEntity saved = repo.save(entity);
    return Market.from(saved);
  }
}
```

## DTO 与校验

```java
public record CreateMarketRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant endDate,
    @NotEmpty List<@NotBlank String> categories) {}

public record MarketResponse(Long id, String name, MarketStatus status) {
  static MarketResponse from(Market market) {
    return new MarketResponse(market.id(), market.name(), market.status());
  }
}
```

## 异常处理

```java
@ControllerAdvice
class GlobalExceptionHandler {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
    String message = ex.getBindingResult().getFieldErrors().stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .collect(Collectors.joining(", "));
    return ResponseEntity.badRequest().body(ApiError.validation(message));
  }

  @ExceptionHandler(AccessDeniedException.class)
  ResponseEntity<ApiError> handleAccessDenied() {
    return ResponseEntity.status(HttpStatus.FORBIDDEN).body(ApiError.of("Forbidden"));
  }

  @ExceptionHandler(Exception.class)
  ResponseEntity<ApiError> handleGeneric(Exception ex) {
    // 记录未预期异常的完整堆栈
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(ApiError.of("Internal server error"));
  }
}
```

## 缓存

需要在配置类上添加 `@EnableCaching`。

```java
@Service
public class MarketCacheService {
  private final MarketRepository repo;

  public MarketCacheService(MarketRepository repo) {
    this.repo = repo;
  }

  @Cacheable(value = "market", key = "#id")
  public Market getById(Long id) {
    return repo.findById(id)
        .map(Market::from)
        .orElseThrow(() -> new EntityNotFoundException("Market not found"));
  }

  @CacheEvict(value = "market", key = "#id")
  public void evict(Long id) {}
}
```

## 异步处理

需要在配置类上添加 `@EnableAsync`。

```java
@Service
public class NotificationService {
  @Async
  public CompletableFuture<Void> sendAsync(Notification notification) {
    // 发送邮件/短信
    return CompletableFuture.completedFuture(null);
  }
}
```

## 日志（SLF4J）

```java
@Service
public class ReportService {
  private static final Logger log = LoggerFactory.getLogger(ReportService.class);

  public Report generate(Long marketId) {
    log.info("generate_report marketId={}", marketId);
    try {
      // 业务逻辑
    } catch (Exception ex) {
      log.error("generate_report_failed marketId={}", marketId, ex);
      throw ex;
    }
    return new Report();
  }
}
```

## 中间件 / 过滤器

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
  private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    long start = System.currentTimeMillis();
    try {
      filterChain.doFilter(request, response);
    } finally {
      long duration = System.currentTimeMillis() - start;
      log.info("req method={} uri={} status={} durationMs={}",
          request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }
  }
}
```

## 分页与排序

```java
PageRequest page = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());
Page<Market> results = marketService.list(page);
```

## 外部调用容错

```java
public <T> T withRetry(Supplier<T> supplier, int maxRetries) {
  int attempts = 0;
  while (true) {
    try {
      return supplier.get();
    } catch (Exception ex) {
      attempts++;
      if (attempts >= maxRetries) {
        throw ex;
      }
      try {
        Thread.sleep((long) Math.pow(2, attempts) * 100L);
      } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw ex;
      }
    }
  }
}
```

## 限流（Filter + Bucket4j）

**安全注意**：`X-Forwarded-For` 请求头默认不可信，因为客户端可以伪造。
仅在以下情况使用转发头：
1. 应用部署在可信的反向代理（nginx、AWS ALB 等）之后
2. 已将 `ForwardedHeaderFilter` 注册为 Bean
3. 已在 application.properties 中配置 `server.forward-headers-strategy=NATIVE` 或 `FRAMEWORK`
4. 反向代理配置为覆盖（而非追加）`X-Forwarded-For` 请求头

正确配置 `ForwardedHeaderFilter` 后，`request.getRemoteAddr()` 将自动从转发头返回正确的客户端 IP。
未配置此功能时，直接使用 `request.getRemoteAddr()` —— 它返回的是直连 IP，这是唯一可信的值。

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  /*
   * 安全说明：此过滤器使用 request.getRemoteAddr() 来识别限流客户端。
   *
   * 如果应用部署在反向代理（nginx、AWS ALB 等）之后，必须正确配置 Spring
   * 以处理转发头，从而准确获取客户端 IP：
   *
   * 1. 在 application.properties/yaml 中设置 server.forward-headers-strategy=NATIVE（云平台）
   *    或 FRAMEWORK
   * 2. 若使用 FRAMEWORK 策略，注册 ForwardedHeaderFilter：
   *
   *    @Bean
   *    ForwardedHeaderFilter forwardedHeaderFilter() {
   *        return new ForwardedHeaderFilter();
   *    }
   *
   * 3. 确保代理覆盖（而非追加）X-Forwarded-For 请求头以防止伪造
   * 4. 为容器配置 server.tomcat.remoteip.trusted-proxies 或等效配置
   *
   * 未配置此功能时，request.getRemoteAddr() 返回的是代理 IP，而非客户端 IP。
   * 不要直接读取 X-Forwarded-For —— 未配置可信代理处理时，该值可被轻易伪造。
   */
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    // 使用 getRemoteAddr()：配置了 ForwardedHeaderFilter 时返回正确客户端 IP，
    // 否则返回直连 IP。在未正确配置代理的情况下，不要信任 X-Forwarded-For 请求头。
    String clientIp = request.getRemoteAddr();

    Bucket bucket = buckets.computeIfAbsent(clientIp,
        k -> Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build());

    if (bucket.tryConsume(1)) {
      filterChain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
    }
  }
}
```

## 后台任务

使用 Spring 的 `@Scheduled` 或集成消息队列（Kafka、SQS、RabbitMQ）。保持处理器幂等性和可观测性。

## 可观测性

- 通过 Logback encoder 输出结构化日志（JSON）
- 指标：Micrometer + Prometheus / OTel
- 链路追踪：Micrometer Tracing + OpenTelemetry 或 Brave 后端

## 生产环境默认配置

- 优先使用构造器注入，避免字段注入
- 启用 `spring.mvc.problemdetails.enabled=true` 以支持 RFC 7807 错误格式（Spring Boot 3+）
- 根据负载配置 HikariCP 连接池大小，设置超时
- 查询使用 `@Transactional(readOnly = true)`
- 通过 `@NonNull` 和 `Optional` 强制空值安全

---

> **记住**：保持 controller 精简、service 专注、repository 简单、异常统一处理。为可维护性和可测试性优化，而非微优化（除非经过验证）。
