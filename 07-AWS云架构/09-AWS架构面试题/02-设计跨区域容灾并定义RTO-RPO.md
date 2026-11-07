# 02. 设计跨区域容灾并定义 RTO/RPO

## 业务分级

按 Criticality 定义 RTO/RPO，而不是所有 Service Active-active。示例：Checkout RTO 15 分钟/RPO 1 分钟，Analytics RTO 24 小时/RPO 12 小时。

## 方案

- Backup/Restore：低成本，高 RTO/RPO。
- Pilot Light：核心 Data/Control 常驻。
- Warm Standby：缩小完整栈。
- Active-active：最低 RTO，最高复杂度。

## 数据

Aurora Global Database、DynamoDB Global Tables、S3 CRR 等各有复制延迟和冲突语义。选择写入 Ownership、Fencing 和一致性。

## 流量

Route 53/Global Accelerator 切流，Health Check 不仅看首页，还验证关键 Dependency。DNS TTL、Existing Connection 和 Client Cache 影响实际 RTO。

## 恢复

IaC 重建、Artifact/Config/Secret/Certificate 跨 Region 可用。Failover Runbook 明确宣布、Fencing、切流、验证；Failback 处理数据同步。

## 演练

定期 Region Game Day，测实际 MTTD、Decision、Restore、Data Loss 和 Customer Communication。用结果更新承诺。

## 面试亮点

RTO/RPO 是业务指标，架构图不构成证明；强调 Data、Control Plane、依赖、Failback 和测试。
