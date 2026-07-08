# OpenIDaaS Gateway 网关服务

## 概述

OpenIDaaS Gateway 是基于 Spring Cloud Gateway 构建的企业级 API 网关服务，提供统一的入口点、路由转发、安全认证、流量控制等功能。

## 主要特性

### 🔐 安全认证
- JWT Token 验证和解析
- 基于角色的访问控制 (RBAC)
- API 密钥认证支持
- CSRF 防护

### 🚦 流量控制
- 多种限流算法支持：
  - 令牌桶算法 (Token Bucket)
  - 漏桶算法 (Leaky Bucket)
  - 固定窗口算法 (Fixed Window)
  - 滑动窗口算法 (Sliding Window)
- 基于客户端IP、用户ID、租户ID的精细化控制
- 白名单机制

### 🔄 路由管理
- 动态路由配置
- 服务发现集成 (Eureka)
- 负载均衡支持
- 路径重写和前缀剥离
- 熔断器集成 (Hystrix)

### 📊 监控运维
- Actuator 健康检查
- Prometheus 指标收集
- 分布式链路追踪
- 详细访问日志记录
- 性能指标统计

### ⚡ 性能优化
- HTTP 响应压缩 (GZIP)
- 连接池优化
- 缓存机制
- 异步非阻塞处理

## 技术栈

- **框架**: Spring Boot 3.5.10, Spring Cloud 2023.0.4
- **网关**: Spring Cloud Gateway
- **服务发现**: Netflix Eureka
- **缓存**: Redis (Lettuce)
- **安全**: Spring Security WebFlux, JWT
- **监控**: Spring Boot Actuator, Micrometer
- **构建**: Maven 3.9+

## 快速开始

### 环境要求
- JDK 21+
- Redis 6.0+
- Eureka Server (可选)

### 启动步骤

1. **克隆项目**
```bash
git clone https://github.com/qoobots/openidaas.git
cd openidaas/openidaas-gateway
```

2. **配置环境**
```yaml
# application.yml
spring:
  redis:
    host: localhost
    port: 6379
    
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

3. **启动服务**
```bash
mvn spring-boot:run
```

4. **验证启动**
```bash
curl http://localhost:8080/actuator/health
```

## 配置详解

### 路由配置
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: auth-service
          uri: lb://openidaas-auth-service
          predicates:
            - Path=/api/auth/**
          filters:
            - StripPrefix=2
            - name: JwtAuthFilter
```

### 限流配置
```yaml
gateway:
  rate-limit:
    algorithm: token-bucket  # 限流算法
    requests-per-second: 10  # 每秒请求数
    burst-capacity: 20       # 突发容量
    window-size-seconds: 60  # 窗口大小
    whitelist-clients:       # 白名单客户端
      - api_key:special-key
```

### 安全配置
```yaml
gateway:
  auth:
    skip-paths:             # 跳过认证的路径
      - /api/auth/login
      - /api/auth/register
      - /health
      - /actuator
```

## 过滤器说明

### 认证过滤器 (`JwtAuthenticationFilter`)
- 验证 JWT Token 的有效性
- 提取用户信息并传递给下游服务
- 支持白名单路径配置

### 限流过滤器 (`RateLimitFilter`)
- 支持四种限流算法
- 基于多种维度的客户端识别
- 灵活的白名单机制

### 追踪过滤器 (`RequestTracingFilter`)
- 生成分布式追踪ID
- 记录完整的请求链路信息
- 支持父子跨度关系

### 压缩过滤器 (`ResponseCompressionFilter`)
- 自动GZIP压缩响应内容
- 基于内容类型和大小的智能判断
- 提升网络传输效率

## API 示例

### 认证请求
```bash
# 登录获取Token
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}'

# 使用Token访问受保护API
curl -X GET http://localhost:8080/api/users/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9..."
```

### 限流响应
```json
{
  "error": "Rate limit exceeded",
  "message": "Too many requests from ip:192.168.1.100: Token bucket empty"
}
```
响应头包含限流信息：
```
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1707580800
Retry-After: 3
```

## 监控端点

| 端点 | 描述 | 访问 |
|------|------|------|
| `/actuator/health` | 健康检查 | 公开 |
| `/actuator/gateway` | 网关路由信息 | 认证 |
| `/actuator/metrics` | 性能指标 | 认证 |
| `/actuator/httptrace` | HTTP追踪 | 认证 |

## 故障排除

### 常见问题

1. **JWT验证失败**
   - 检查JWT密钥配置是否正确
   - 验证Token是否过期
   - 确认请求头格式正确

2. **限流触发频繁**
   - 调整限流参数配置
   - 检查Redis连接状态
   - 审核白名单配置

3. **路由转发失败**
   - 确认下游服务是否注册到Eureka
   - 检查服务名称是否匹配
   - 验证网络连通性

### 日志级别调整
```yaml
logging:
  level:
    com.qoobot.openidaas.gateway: DEBUG
    org.springframework.cloud.gateway: INFO
```

## 性能调优

### JVM 参数推荐
```bash
-Xms2g -Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
```

### 连接池配置
```yaml
spring:
  redis:
    lettuce:
      pool:
        max-active: 50
        max-idle: 20
        min-idle: 10
```

## 贡献指南

欢迎提交 Issue 和 Pull Request 来帮助我们改进项目！

### 开发环境设置
1. Fork 项目
2. 创建功能分支
3. 编写测试用例
4. 提交 Pull Request

## 许可证

Apache License 2.0

---
**OpenIDaaS** - 企业级身份认证即服务解决方案