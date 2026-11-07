# 05. Incident Management、Runbook 与自动修复

## Incident Role

Incident Commander 负责决策与节奏；Operations 执行；Communications 对内外同步；Scribe 记录 Timeline。角色减少多人无协调操作。

## 流程

Detect → Triage → Declare → Mitigate → Recover → Validate → Postmortem。优先恢复用户服务，再做完整 Root Cause。

## Runbook

包含 Trigger、Impact、Dashboard、诊断、受控 Mitigation、Rollback、Escalation 和恢复验证。命令明确 Scope，危险动作有 Preview/Approval。

## 自动修复

适合重复、低风险、可逆动作，如替换不健康实例、扩容 Consumer。设置 Guardrail、Rate、Max Action、Audit 和 Kill Switch。不要自动重试不可逆数据操作。

## AWS 工具

CloudWatch Alarm/EventBridge → Lambda/SSM Automation；Systems Manager Incident Manager（或组织平台）协调；CloudTrail 保留操作证据。

## Postmortem

Blameless 分析 Contributing Factor，Action 有 Owner/Date/验证。更新 Runbook、Alarm、IaC、Chaos Test 和 Architecture Standard。

## 总结

Incident Management 是技术与协作系统。清晰角色、可执行 Runbook 和受控自动化共同降低 MTTR。
