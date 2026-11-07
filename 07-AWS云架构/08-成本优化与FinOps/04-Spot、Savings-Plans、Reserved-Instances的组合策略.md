# 04. Spot、Savings Plans、Reserved Instances 的组合策略

## 负载分层

把使用分为稳定 Baseline、可预测波动、不可预测 Burst 和可中断 Batch。

```text
Stable baseline -> Savings Plans / RI
Variable online -> On-demand + Auto Scaling
Interruptible -> Spot
```

## Commitment

基于保守持续使用购买，避免把短期峰值承诺多年。比较 1/3 年、付款方式和灵活性。持续观察 Coverage 与 Utilization。

## Spot

多 Instance Type/AZ、Capacity-optimized、Checkpoint、Queue 和中断处理。Spot Savings 要扣除重试、延迟和工程成本。

## RI

RDS/OpenSearch 等服务的 Reserved Capacity 规则不同，逐项评估。EC2 场景常优先 Savings Plans 的灵活性，但高稳定特定配置可比较 RI。

## 风险

架构迁移到 Serverless/Graviton/不同 Region 可能让承诺闲置。购买前与 Roadmap、增长和合同一起 Review。

## 治理

Central FinOps 管 Portfolio，Team 提供 Forecast 与变更信息；使用 Allocation 让折扣收益透明分配。

## 总结

折扣策略是容量与财务组合，不是一次购买。保守覆盖 Baseline、Spot 承担可恢复工作，并持续调整。
