# 03. GitOps、Argo CD、期望状态与漂移检测

## GitOps 原则

Git 保存声明式期望状态；Agent Pull 并持续 Reconcile；变更通过 Pull Request；实际状态偏离时检测或自动修复。

## Argo CD

监控 Git 与 Kubernetes，展示 Sync/Health/Diff，支持 Automated Sync、Wave/Hook、Multi-cluster 和 Rollback。它部署 Manifest，不替代 Build/Test/Security。

## Repository

分离 Application Source 与 Environment Config 可减少发布权限耦合。Image Automation 通过 PR 更新 Digest，而不是使用浮动 `latest`。

## Secret

Git 中不放明文。使用 External Secrets、SOPS/Sealed Secrets 等，并管理 Key、Rotation 和人员权限。

## Drift

生产手工修改会被检测；是否 Auto-heal 取决于风险。Emergency Change 也应尽快回写 Git，否则 Reconcile 会覆盖或长期漂移。

## Promotion

同一 Image Digest 通过环境 PR Promote。配置 Diff 可审计，Canary 可由 Argo Rollouts 等 Progressive Delivery Controller 处理。

## 风险

- Git/Control Plane 权限成为关键攻击面。
- 错误 Commit 可快速扩散多个 Cluster。
- Controller Version/CRD 兼容。
- Git 不适合保存 Runtime Secret 和高频状态。

## 总结

GitOps 把交付变成可审计 Reconciliation。严格权限、不可变 Artifact、Secret 方案和渐进推广防止自动化放大错误。
