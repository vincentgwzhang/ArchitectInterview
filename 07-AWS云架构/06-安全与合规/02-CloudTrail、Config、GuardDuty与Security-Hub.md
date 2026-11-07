# 02. CloudTrail、Config、GuardDuty 与 Security Hub

## CloudTrail

记录 AWS API Activity。Organization Trail 集中到 Log Archive Account，启用完整 Region、Log Validation 和受控 S3。Data Event（如 S3 Object/Lambda）量大且付费，按风险选择。

## AWS Config

记录 Resource Configuration 变化并按 Rule 评估合规。用于 Drift、Public Resource、Encryption 和 Tag 等；它不是实时攻击检测。

## GuardDuty

分析 CloudTrail、VPC DNS/Flow、EKS 等信号发现 Credential、Network 和 Malware 威胁。Finding 需与资产 Criticality 和 Context Triage。

## Security Hub

聚合 Security Finding、标准检查与多账户状态，通过 ASFF 统一格式。它不是 SIEM 全替代，但可作为 Cloud Security Posture 中心。

## Multi-account

使用 Delegated Administrator 集中启用，成员 Account 不能轻易关闭。EventBridge 将高危 Finding 路由到 Ticket/SIEM/自动响应。

## 响应

自动隔离 Role/Instance 前设置 Guardrail，避免误报扩大事故。保留 Evidence、Runbook、Owner 和 SLA。

## 总结

CloudTrail 回答谁做了什么，Config 记录配置，GuardDuty 检测威胁，Security Hub 汇总治理。关键是集中、不可篡改和可行动流程。
