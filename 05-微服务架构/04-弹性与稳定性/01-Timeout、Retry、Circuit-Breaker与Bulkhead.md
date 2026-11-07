# 01. Timeout、Retry、Circuit Breaker 与 Bulkhead

## Timeout

每个远程调用设置连接/响应 Timeout，并从端到端 Deadline 分配预算。超时只说明 Caller 未得到结果，不证明下游未执行。

## Retry

仅对明确 Transient 且 Idempotent 操作重试。使用 Exponential Backoff、Jitter、Max Attempts 和总 Deadline。多个 Service 各重试 3 次可能形成指数流量，应传播 Retry Context/Budget。

## Circuit Breaker

Closed 正常调用；错误/慢调用超过阈值进入 Open 快速失败；Half-open 探测恢复。按 Dependency 与 Operation 粒度隔离，避免一个接口问题切断整个供应商。

## Bulkhead

为不同 Dependency、Tenant 或 Workload 隔离 Thread/Connection/Concurrency/Queue。一个慢报表任务不能耗尽 Checkout 资源。

## 组合顺序

典型：Deadline → Bulkhead/Rate Limit → Circuit Breaker → Timeout → 受控 Retry。实际由 Library 语义决定，必须测试失败路径。

## Fallback

只有业务允许时使用 Stale Cache、Default 或 Degraded Feature。Fallback 也可能失败，并需 Metric 与告警。

## 总结

这些 Pattern 用于限制故障传播。错误配置会放大流量或隐藏事故，因此必须结合 SLO、幂等和容量测试。
