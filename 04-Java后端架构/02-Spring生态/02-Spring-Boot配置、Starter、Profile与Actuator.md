# 02. Spring Boot 配置、Starter、Profile 与 Actuator

## Externalized Configuration

配置来源具有优先级：文件、Environment、Command Line、Config Server 等。使用 `@ConfigurationProperties` + Validation 替代散落的 `@Value`。

```java
@ConfigurationProperties("payment")
@Validated
record PaymentProperties(@NotBlank URI endpoint, @Positive Duration timeout) {}
```

启动时 Fail Fast，避免生产第一次调用才发现配置缺失。

## Profile

Profile 适合选择环境或能力组合，但大量 `dev/test/prod` 条件会让行为难以推理。推荐同一 Artifact 在不同环境使用外部配置，Secret 由 Secret Manager 提供。

Browser/Repository 中不要保存生产 Secret。

## Starter

内部 Starter 可统一 Observability、Security、HTTP Client 和 Resilience 默认值。必须允许受控覆盖，并避免把所有组织逻辑塞进一个巨大 Starter，导致升级 Blast Radius。

## Actuator

提供 Health、Metrics、Info、Environment 等管理 Endpoint。生产只暴露必要 Endpoint，管理端口与 Network/Authentication 受控，敏感值 Sanitized。

- Liveness：进程是否需要重启，不依赖共享外部系统。
- Readiness：是否能接收流量，可反映关键初始化状态。

错误 Health 设计会在数据库故障时同时重启所有 Pod，放大事故。

## 配置治理

- Schema/Validation。
- Version 与 Owner。
- Audit Change。
- Feature Flag 与普通配置分离。
- Dynamic Config 有 Rollback 和安全边界。

## 总结

Spring Boot 配置应类型化、可验证、外部化；Starter 提供 Golden Path；Actuator 支持安全运营，而不是无条件暴露内部状态。
