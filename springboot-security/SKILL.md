---
name: springboot-security
description: Java Spring Boot 服务的 Spring Security 最佳实践：认证/授权、输入校验、CSRF、密钥管理、安全响应头、限流及依赖安全。
origin: ECC
---

# Spring Boot 安全规范

在添加认证、处理输入、创建接口或涉及密钥时启用。

## 何时启用

- 添加认证（JWT、OAuth2、会话制）
- 实现授权（@PreAuthorize、基于角色的访问控制）
- 校验用户输入（Bean Validation、自定义校验器）
- 配置 CORS、CSRF 或安全响应头
- 管理密钥（Vault、环境变量）
- 添加限流或防暴力破解
- 扫描依赖中的 CVE

## 认证

- 优先使用无状态 JWT 或带撤销列表的不透明令牌
- 会话使用 `httpOnly`、`Secure`、`SameSite=Strict` Cookie
- 通过 `OncePerRequestFilter` 或资源服务器验证令牌

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
  private final JwtService jwtService;

  public JwtAuthFilter(JwtService jwtService) {
    this.jwtService = jwtService;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String header = request.getHeader(HttpHeaders.AUTHORIZATION);
    if (header != null && header.startsWith("Bearer ")) {
      String token = header.substring(7);
      Authentication auth = jwtService.authenticate(token);
      SecurityContextHolder.getContext().setAuthentication(auth);
    }
    chain.doFilter(request, response);
  }
}
```

## 授权

- 启用方法级安全：`@EnableMethodSecurity`
- 使用 `@PreAuthorize("hasRole('ADMIN')")` 或 `@PreAuthorize("@authz.canEdit(#id)")`
- 默认拒绝；仅暴露必需的权限范围

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

  @PreAuthorize("hasRole('ADMIN')")
  @GetMapping("/users")
  public List<UserDto> listUsers() {
    return userService.findAll();
  }

  @PreAuthorize("@authz.isOwner(#id, authentication)")
  @DeleteMapping("/users/{id}")
  public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
  }
}
```

## 输入校验

- 在 Controller 上使用 Bean Validation + `@Valid`
- 在 DTO 上应用约束：`@NotBlank`、`@Email`、`@Size`、自定义校验器
- 渲染前对任何 HTML 使用白名单过滤

```java
// ❌ 不推荐：未校验
@PostMapping("/users")
public User createUser(@RequestBody UserDto dto) {
  return userService.create(dto);
}

// ✅ 推荐：校验 DTO
public record CreateUserDto(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @Min(0) @Max(150) Integer age
) {}

@PostMapping("/users")
public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserDto dto) {
  return ResponseEntity.status(HttpStatus.CREATED)
      .body(userService.create(dto));
}
```

## SQL 注入防护

- 使用 Spring Data Repository 或参数化查询
- 原生查询使用 `:param` 绑定；禁止字符串拼接

```java
// ❌ 不推荐：原生查询中的字符串拼接
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)

// ✅ 推荐：参数化原生查询
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);

// ✅ 推荐：Spring Data 派生查询（自动参数化）
List<User> findByEmailAndActiveTrue(String email);
```

## 密码加密

- 始终使用 BCrypt 或 Argon2 哈希密码 —— 禁止明文存储
- 使用 `PasswordEncoder` Bean，而非手动哈希

```java
@Bean
public PasswordEncoder passwordEncoder() {
  return new BCryptPasswordEncoder(12); // cost factor 12
}

// 在 service 中
public User register(CreateUserDto dto) {
  String hashedPassword = passwordEncoder.encode(dto.password());
  return userRepository.save(new User(dto.email(), hashedPassword));
}
```

## CSRF 防护

- 浏览器会话应用：保持 CSRF 启用；在表单/请求头中包含令牌
- 纯 API + Bearer 令牌：禁用 CSRF，依赖无状态认证

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

## 密钥管理

- 源代码中禁止包含密钥；从环境变量或 Vault 加载
- `application.yml` 中不含凭证；使用占位符
- 定期轮换令牌和数据库凭证

```yaml
# ❌ 不推荐：硬编码在 application.yml 中
spring:
  datasource:
    password: mySecretPassword123

# ✅ 推荐：环境变量占位符
spring:
  datasource:
    password: ${DB_PASSWORD}

# ✅ 推荐：Spring Cloud Vault 集成
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      token: ${VAULT_TOKEN}
```

## 安全响应头

```java
http
  .headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
      .policyDirectives("default-src 'self'"))
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::sameOrigin)
    .xssProtection(Customizer.withDefaults())
    .referrerPolicy(rp -> rp.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER)));
```

## CORS 配置

- 在安全过滤器层配置 CORS，而非每个 Controller 分别配置
- 限制允许的来源 —— 生产环境禁止使用 `*`

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
  CorsConfiguration config = new CorsConfiguration();
  config.setAllowedOrigins(List.of("https://app.example.com"));
  config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
  config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
  config.setAllowCredentials(true);
  config.setMaxAge(3600L);

  UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/api/**", config);
  return source;
}

// 在 SecurityFilterChain 中：
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

## 限流

- 在高代价接口上应用 Bucket4j 或网关级限流
- 对突发流量记录日志并告警；返回 429 并附带重试提示

```java
// 使用 Bucket4j 进行按接口限流
@Component
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  private Bucket createBucket() {
    return Bucket.builder()
        .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
        .build();
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String clientIp = request.getRemoteAddr();
    Bucket bucket = buckets.computeIfAbsent(clientIp, k -> createBucket());

    if (bucket.tryConsume(1)) {
      chain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
      response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
    }
  }
}
```

## 依赖安全

- 在 CI 中运行 OWASP Dependency Check / Snyk
- 保持 Spring Boot 和 Spring Security 为受支持版本
- 发现已知 CVE 时构建失败

## 日志与个人隐私信息（PII）

- 禁止记录密钥、令牌、密码或完整银行卡号
- 脱敏敏感字段；使用结构化 JSON 日志

## 文件上传

- 校验大小、内容类型和扩展名
- 存储在 Web 根目录之外；必要时进行扫描

## 发布前检查清单

- [ ] 认证令牌验证正确且过期设置正确
- [ ] 每个敏感路径都有授权守卫
- [ ] 所有输入均已校验和过滤
- [ ] 无字符串拼接的 SQL
- [ ] CSRF 姿态与应用类型匹配
- [ ] 密钥已外部化；无提交
- [ ] 安全响应头已配置
- [ ] API 已限流
- [ ] 依赖已扫描且为最新版本
- [ ] 日志中无敏感数据

---

> **记住**：默认拒绝，校验输入，最小权限，安全配置优先。
