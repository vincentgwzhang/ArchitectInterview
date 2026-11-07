# 02. Rightsizing、自动伸缩与闲置资源治理

## Rightsizing

基于 P95 CPU/Memory/Network/IO、SLO 和增长调整 EC2/RDS/ECS Request。只看平均 CPU 可能忽略 Memory、Burst 和故障容量。

Compute Optimizer 可提供建议，但需结合业务 Seasonality、License、Instance Family 和迁移成本。

## Auto Scaling

Scale-out 保护性能，Scale-in 节省成本。选择正确 Metric、Warmup、Min/Max 和 Scheduled Scaling。确保下游不会因实例增加而超载。

## 闲置治理

常见浪费：未挂 EBS、旧 Snapshot/AMI、Idle Load Balancer/NAT、Elastic IP、低使用 RDS、过期 Preview、测试集群全天运行。

建立 Owner、TTL Tag、通知、Grace Period 和自动 Stop/Delete。删除前验证 Backup/Dependency，并优先可恢复动作。

## 非生产

按工作时间 Schedule Stop，使用较小规格/Spot，Preview 自动过期。某些托管服务停机限制需纳入方案。

## Guardrail

成本优化不能破坏 Multi-AZ、Backup、Security 和 SLO。记录 Savings、Risk 和回滚。

## 总结

Rightsizing 是持续过程。准确 Telemetry、Auto Scaling 和有 Owner 的闲置清理能降低成本，同时保护可靠性。
