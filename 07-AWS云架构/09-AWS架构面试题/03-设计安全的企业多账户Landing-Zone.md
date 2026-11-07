# 03. 设计安全的企业多账户 Landing Zone

## 组织结构

Organizations + Control Tower，OU 分 Security、Infrastructure、Workload Prod/Non-prod、Sandbox。Management Account 不运行 Workload。

## 核心账户

- Log Archive：Organization CloudTrail/Config Log，不可篡改。
- Security Tooling：GuardDuty/Security Hub delegated admin、SIEM。
- Network：TGW、DNS、Firewall/Endpoint。
- Shared Services：Artifact、Directory、CI Platform。

## 身份

Identity Center 联邦企业 IdP，Permission Set、MFA、短 Session、Break-glass。Workload/CI 使用 Role/OIDC，不创建长期 Key。

## Guardrail

SCP 禁止关闭日志、使用未批准 Region、公开敏感资源和越权动作；Config/Policy as Code 检测加密、Tag、Backup。例外有期限。

## Account Vending

自动创建 Account、Baseline VPC、Log、Budget、Contact、Role 和 IaC State。Owner/Cost Center 必填，生命周期可关闭/归档。

## 网络与数据

Hub-spoke/TGW 或去中心化网络按需求；Private Endpoint、DNS、Egress、KMS、Backup Account 和 Data Residency。

## 面试亮点

Landing Zone 是持续产品，不是一次脚本。说明升级、Account Lifecycle、Evidence、Platform Ownership 和 Team Autonomy。
