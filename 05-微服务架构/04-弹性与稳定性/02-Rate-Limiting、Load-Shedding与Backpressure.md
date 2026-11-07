# 02. Rate Limiting、Load Shedding 与 Backpressure

## Rate Limiting

保护公平性、配额和容量。Token Bucket 允许 Burst；Leaky Bucket 平滑；Sliding Window 更精确但状态成本较高。

限制维度：User、Tenant、API、IP、Cost Unit、Token/Message。返回 429、Retry-After 和配额信息，但不要泄露安全敏感阈值。

## Load Shedding

系统已经接近 Saturation 时主动拒绝低价值工作，保持核心 SLO：

- 丢弃可重建 Analytics。
- 暂停 Batch。
- 关闭 Recommendation。
- 限制昂贵 Query。
- 使用 Admission Control。

快速明确失败优于长时间排队后超时。

## Backpressure

Consumer 告诉 Producer 自己能处理多少。实现方式包括 Bounded Queue、Reactive Demand、Semaphore、Consumer Lag Auto Scaling 与 Producer Quota。

Queue 只能延迟压力，不能消除压力。监控 Message Age 比单纯 Depth 更能表达用户影响。

## Priority

按 Business Criticality 分 Queue/Pool，防止低优先级任务占用全部资源。要防止低优先级永久 Starvation。

## Distributed Limit

集中计数准确但增加 Latency/Dependency；Local Limit 快但全局不精确。根据配额严谨度选择，并考虑 Redis/Limit Service 故障时 Fail-open 或 Fail-closed。

## 总结

限流控制入口，Backpressure 传播处理能力，Load Shedding 在过载时保护核心功能。三者共同维持稳定退化。
