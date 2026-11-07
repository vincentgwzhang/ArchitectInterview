# 02. AWS Organizations、Control Tower 与多账户策略

## 为什么 Multi-account

Account 是强隔离、配额、账单和权限边界。按 Production/Non-production、Business Unit、Workload、Security/Log Archive 划分，比所有资源放一个 Account 更安全、可管理。

## Organizations

提供组织层级、Consolidated Billing、Service Control Policy（SCP）和账户管理。SCP 定义最大可用权限，不直接授予权限；显式 Deny 会覆盖成员账户 IAM Allow。

## Control Tower

基于 Organizations 建立 Landing Zone、账户工厂、日志、安全基线与 Guardrail。它加速标准化，但自定义 Control、升级和现有账户导入仍需治理。

## 常见账户

```text
Management
Security Tooling
Log Archive
Network/Shared Services
Workload Prod
Workload Non-prod
Sandbox
```

Management Account 不运行普通 Workload，并严格限制人员访问。

## Guardrail

使用 SCP、AWS Config、CloudTrail、Identity Center 和 IaC。Preventive Control 阻止动作；Detective Control 发现偏离。例外要有 Owner、期限和补偿控制。

## 总结

Multi-account 是 AWS 企业治理基础。Organizations 定义组织边界，Control Tower 提供 Landing Zone，Workload Team 在 Guardrail 内自治。
