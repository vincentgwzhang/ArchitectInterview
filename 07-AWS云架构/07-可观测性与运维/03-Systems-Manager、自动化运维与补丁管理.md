# 03. Systems Manager、自动化运维与补丁管理

## Systems Manager

提供 Fleet Manager、Run Command、Session Manager、Patch Manager、Automation、Parameter Store 和 Inventory 等能力。

## Session Manager

无需开放 SSH/RDP Inbound 或管理 Bastion Key，通过 IAM、日志和审计访问 Instance。限制 Session Role、Command 和 Data Channel，生产 Break-glass 需审批。

## Run Command/Automation

批量执行受控 Document，支持 Parameter、Approval、Rate/Error Threshold。Automation Runbook 用于 AMI、Recovery、Patch 和 Incident；必须 Idempotent、可停止并记录输出。

## Patch

定义 Patch Baseline、Maintenance Window、分组和 Reboot 策略。先 Canary Account/Instance，再 Wave 推广；监控 Compliance 和失败。

Immutable Workload 更适合重建新 AMI/Image 后替换，而非长期原地 Patch。两种模式可按 Workload 使用。

## 安全

SSM Agent/Endpoint、Instance Role 最小权限，Command Output 加密，禁止任意人员运行任意 Shell。使用 VPC Endpoint 减少公网访问。

## 总结

Systems Manager 将 Fleet 操作、访问与 Patch 自动化并审计。自动化仍需 Wave、权限、Rollback 和 Immutable 策略。
