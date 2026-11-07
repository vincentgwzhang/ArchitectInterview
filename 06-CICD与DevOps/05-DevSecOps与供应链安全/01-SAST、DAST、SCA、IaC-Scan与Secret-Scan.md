# 01. SAST、DAST、SCA、IaC Scan 与 Secret Scan

## 工具定位

- SAST：分析 Source/Bytecode 中潜在漏洞。
- DAST：从运行外部测试应用行为。
- SCA：识别第三方依赖、License 和 CVE。
- IaC Scan：检查 Cloud/Kubernetes 配置。
- Secret Scan：检测 Credential/Token Pattern。

没有单一 Scanner 覆盖所有风险。

## Pipeline

PR 运行快速增量 SAST/SCA/Secret/IaC；主干和 Staging 运行完整 Scan/DAST；生产持续监控新 CVE 与 Drift。

## Triage

按 Severity、Reachability、Exploitability、Exposure、Asset Criticality 和 Fix Availability 排序。CVSS 高不一定真实可达，但不能在无证据时忽略。

## Gate

新 Critical/High、泄露 Secret 和公开敏感资源通常阻塞。既有 Debt 进入 SLA 和 Owner；不要让所有 Warning 阻塞导致团队绕过工具。

## DAST 安全

在隔离环境、测试账户和允许范围执行，避免破坏生产数据。生产使用被动/安全检查需明确授权。

## Secret

检测到真实 Secret 时立即撤销/轮换，即使 Commit 尚未 Merge。仅删除字符串不够，因为 Log/History/Cache 可能已复制。

## 总结

DevSecOps Scan 是持续风险反馈。组合覆盖、风险型门禁、可执行修复和 SLA 比追求零告警更有效。
