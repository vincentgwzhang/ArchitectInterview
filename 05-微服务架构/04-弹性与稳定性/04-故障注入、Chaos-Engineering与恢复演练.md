# 04. 故障注入、Chaos Engineering 与恢复演练

## 目标

Chaos Engineering 用受控实验验证系统在故障下是否保持 Steady State，而不是随机破坏生产。

## 流程

1. 定义 Steady-state Metric/SLO。
2. 提出 Hypothesis。
3. 选择最小 Blast Radius。
4. 设置 Abort Condition 和 Kill Switch。
5. 注入故障。
6. 观察 Detection、Degradation、Recovery。
7. 记录 Gap 并修复。

## 实验类型

- Pod/Node/AZ Failure。
- Network Latency/Loss/Partition。
- Dependency 5xx/Timeout。
- Kafka Lag/Broker Loss。
- Disk Full/CPU/Memory Pressure。
- Credential/Certificate Expiry。
- DNS/Control Plane Failure。

## 安全

先在 Test/Staging 和低风险生产 Slice 开始。明确 Owner、时间窗、客户影响、Rollback 和沟通。不要在已知系统不稳定时进行无关实验。

## Game Day

不仅验证技术，还验证 Alert、On-call、权限、Runbook、Vendor Escalation 和业务决策。记录实际 MTTD/MTTR 与 RTO/RPO。

## 常见错误

- 只证明系统会失败，不验证假设。
- 没有自动恢复 Metric。
- 发现问题后无 Owner。
- 只杀 Pod，从不测试数据/依赖/人类流程。

## 总结

Chaos 是可靠性验证工具。小范围、可停止、可测量和有修复闭环的实验才能提升韧性。
