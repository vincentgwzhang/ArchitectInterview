# 04. Graviton、Spot、Reserved Instances 与 Savings Plans

## Graviton

ARM-based Processor，常有更好 Price/Performance。Java/容器通常迁移容易，但需验证 Native Library、Agent、Image Architecture 和性能。用 Multi-arch Image 支持渐进迁移。

## Spot

使用 AWS 闲置容量，价格低但可被中断。适合 Stateless、Batch、CI、可重试 Consumer。使用多 Instance Type/AZ、Capacity-optimized、Checkpoint 和优雅终止。

关键不可中断 Base Load 不应全部依赖 Spot。

## Reserved Instances

对特定 EC2/RDS 使用承诺换折扣，灵活性取决于类型。适合稳定已知使用。

## Savings Plans

按每小时计算使用承诺，Compute Savings Plans 对 EC2/Fargate/Lambda 更灵活；EC2 Instance Savings Plans 折扣更高但限制更多。

## 组合

```text
Baseline -> Savings Plans/RI
Variable predictable -> On-demand
Fault-tolerant burst -> Spot
```

承诺基于保守 Baseline，避免过度购买。持续看 Coverage、Utilization 和未来架构变化。

## 总结

Graviton 优化单位性能，Spot 利用可中断容量，RI/Savings Plans 优化稳定使用。都需以真实 Workload 与可恢复性验证。
