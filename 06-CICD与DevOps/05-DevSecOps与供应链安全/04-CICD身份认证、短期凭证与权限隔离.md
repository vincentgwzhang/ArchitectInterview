# 04. CI/CD 身份认证、短期凭证与权限隔离

## 威胁

Pipeline 可修改生产，是高价值攻击面。风险包括长期 Cloud Key、恶意 PR、Dependency/Action 被劫持、Runner Persistence 和 Log 泄密。

## Federated Identity

CI 通过 OIDC 向 Cloud STS 换取短期 Role。Trust Policy 限制 Repository、Branch/Tag、Workflow、Environment 和 Audience。

不要只验证“来自 GitHub/GitLab”，否则其他 Repository 可能获得权限。

## 权限隔离

- PR Build 无生产 Secret/Deploy 权限。
- Build 与 Deploy Identity 分离。
- 环境独立 Role/Account。
- Production 需要受保护 Environment/Approval（按风险）。
- Artifact Promotion 不重新 Build。

## Runner

不可信代码使用 Ephemeral Isolated Runner。清理 Workspace/Credential，限制 Network/Egress，固定第三方 Action/Plugin Digest。

## Secret

优先消除长期 Secret。必要 Secret 存 Manager，Mask Log，限制 Fork/Debug 和 Artifact 上传。发现泄露立即 Rotate。

## Audit

记录谁触发、哪个 Commit、哪个 Workflow/Artifact、获得哪个 Role、部署何处和审批人。异常 Branch/时间/环境触发告警。

## 总结

CI/CD 应采用零信任：短期身份、上下文约束、职责分离、隔离 Runner 和完整 Provenance，防止供应链入口直达生产。
