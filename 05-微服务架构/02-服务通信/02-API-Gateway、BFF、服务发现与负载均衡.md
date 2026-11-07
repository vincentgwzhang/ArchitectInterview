# 02. API Gateway、BFF、服务发现与负载均衡

## API Gateway

统一处理 TLS、Routing、Authentication、Rate Limit、Request Size、WAF、Observability 和 Version。不要把核心业务编排全部放入 Gateway，否则成为难测试的 Central Monolith。

## BFF

Backend for Frontend 为 Web、Mobile 等特定体验聚合 API，减少 Chatty Call 并适配数据形状。BFF 应拥有 Presentation Logic，不复制 Domain Rule 或绕过下游 Ownership。

## 服务发现

- Client-side：Client 从 Registry 获取实例并选择。
- Server-side：Load Balancer/Service Mesh 选择实例。
- Kubernetes DNS/Service 提供内建 Discovery。

需要处理实例注册延迟、健康状态、DNS Cache 和 Graceful Deregistration。

## 负载均衡

算法：Round Robin、Least Connections、Weighted、Consistent Hash。选择需考虑长连接、实例容量、Session 和 Cache Locality。

Health Check 区分 Liveness/Readiness。实例 Shutdown 前先停止新流量并完成 In-flight Request。

## Resilience

Gateway 是关键入口，需 Multi-AZ、无状态扩容和配置回滚。设置全局与租户限流，但业务级配额仍由 Domain Service 验证。

## Security

Gateway Authentication 不替代服务内 Authorization。内部调用传播可信身份/Scope，避免未经验证 Header 被伪造。

## 总结

Gateway 管理边缘策略，BFF 管理客户端体验，Discovery/LB 管理动态实例。职责清晰才能避免新的单点和中央耦合。
