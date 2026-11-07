# 03. IAM User、Role、Policy、STS 与最小权限

## 身份

- IAM User：长期身份，应尽量避免用于人员和 Workload。
- IAM Role：由用户、服务或外部身份临时 Assume。
- STS：签发短期 Credential。
- IAM Identity Center：集中 Workforce Access。

人员使用 Federation/SSO，应用使用 IAM Role/Workload Identity，CI 使用 OIDC Assume Role。

## Policy

Identity Policy、Resource Policy、Permission Boundary、SCP、Session Policy 共同决定权限。显式 Deny 优先。Policy 包含 Action、Resource、Condition，尽量不用 `*`。

## Least Privilege

从任务所需权限开始，使用 CloudTrail/Access Analyzer 观察实际调用后收紧。通过 Tag、Source VPC、Organization ID、MFA 等 Condition 限制。

## Role Trust

Trust Policy 同样关键。Cross-account Role 限定 Principal、External ID/Organization 和 Session，防 Confused Deputy。

## 运营

短 Session、Key Rotation、Access Review、Unused Permission、Break-glass 和完整 Audit。禁止共享 Root；Root 开 MFA 并只用于极少任务。

## 总结

IAM 是授权图而非单一 Policy。优先临时身份，细分 Action/Resource/Condition，并持续验证实际使用。
