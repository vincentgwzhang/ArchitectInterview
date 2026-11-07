# 05. Shared Responsibility Model 与 Well-Architected Framework

## Shared Responsibility

AWS 负责“Security of the Cloud”，客户负责“Security in the Cloud”。责任随服务模型变化：EC2 中客户管理 OS/Patch；RDS 中 AWS 管底层 OS，但客户仍负责账号、网络、Schema、Backup Policy 和数据。

Managed Service 减少部分运营责任，不转移业务数据和权限责任。

## Well-Architected Pillars

- Operational Excellence。
- Security。
- Reliability。
- Performance Efficiency。
- Cost Optimization。
- Sustainability。

Review 以 Workload 和业务 Context 为单位，识别 High-risk Item、Owner 和改进 Roadmap，不是一次性合规打勾。

## Trade-off

例如 Multi-region 提高 Reliability，却增加 Cost、Operational Complexity 和数据复制。Framework 帮助系统讨论，而不是给出唯一架构。

## 实践

在 Design、Launch、重大 Change 和 Incident 后 Review；把结论转为 ADR、Backlog、SLO 和自动 Guardrail。使用真实 Metric 验证改进。

## 总结

Shared Responsibility 明确云与客户边界，Well-Architected 用多维视角审查 Workload。最终责任仍需落到具体团队与可验证控制。
