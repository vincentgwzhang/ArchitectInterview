# 05. Policy as Code、合规检查与基础设施测试

## Policy as Code

把 Security/Compliance/Cost Guardrail 写成可执行规则，例如：禁止公开 Storage、要求 Encryption、限制 Region、要求 Tag 和 Backup。

执行层：

- IaC 静态 Scan。
- Terraform Plan Policy。
- Kubernetes Admission。
- Cloud Runtime Config Scan。
- GitOps Diff。

## 测试金字塔

- Unit：Module/Function。
- Static：Format、Validate、Lint、Security。
- Plan/Synth：资源 Diff 与 Policy。
- Integration：临时环境实际创建。
- Runtime：Connectivity、Encryption、Failover、Compliance。

## Policy 设计

规则有 Owner、Reason、Severity、Remediation 和版本。先 Audit Mode 观察影响，再 Enforcement。避免无法解释的大量 False Positive。

## Exception

例外记录 Business Justification、Risk Owner、Compensating Control 和 Expiry。Policy Engine 识别批准例外，但持续显示与复审。

## Compliance Evidence

自动保存 Plan、Approval、Scan、Deployment Identity、Runtime Check 和 Change Log，减少人工截图。Evidence 本身需防篡改与保留策略。

## Drift

IaC 通过不代表 Runtime 永远合规。持续 Cloud Config/Admission 检测并自动或人工修复。

## 总结

Policy as Code 把标准变为持续 Guardrail。好的策略可解释、可测试、可例外并连接 Runtime Evidence。
