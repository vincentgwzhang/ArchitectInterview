# 04. Secret 管理、密钥轮换与最小权限

## Secret Lifecycle

创建 → 分发 → 使用 → 轮换 → 撤销 → 审计。Secret 不应进入 Source、Image、普通 CI Log、Chat 或 Ticket。

## Workload Identity

优先 OIDC、IAM Role、Managed Identity 和短期 Token，避免长期 Access Key。身份绑定 Workload、Environment 和最小 Scope。

## Secret Manager

集中加密、Version、Access Policy、Rotation 和 Audit。应用通过 Runtime 身份读取，Cache 需短时并支持轮换。Secret Manager 不应成为每请求同步依赖。

## Rotation

安全轮换通常双版本：创建新 Credential → 系统接受新旧 → 应用切换 → 验证 → 撤销旧。数据库密码可用动态 Credential 或 Proxy 减少中断。

## CI/CD

CI 使用 Federated Identity 获取短期部署权限；PR from Fork 不获得 Secret。环境保护、防输出 Masking 和受控 Runner。

## Least Privilege

从任务所需 Action/Resource/Condition 建 Policy。定期 Access Review、分析未使用权限、设置 Permission Boundary。Break-glass 权限短期、审批并完整审计。

## 泄露响应

立即撤销/轮换、查 Audit、识别受影响 Resource、保存 Evidence、修复来源。仅从 Git 删除不能消除历史泄露。

## 总结

最好的 Secret 是不创建长期 Secret。Workload Identity、短期授权、自动轮换和细粒度审计构成安全基础。
