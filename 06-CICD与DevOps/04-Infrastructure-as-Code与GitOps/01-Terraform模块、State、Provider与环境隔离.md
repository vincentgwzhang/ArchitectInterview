# 01. Terraform 模块、State、Provider 与环境隔离

## 声明式模型

Terraform 比较 Configuration 与 State/Remote API，生成 Plan 并 Apply。Plan 需要 Review，尤其关注 Destroy、Replace、权限和网络变化。

## Module

Module 应封装稳定能力，如 VPC、Service、Database，而不是简单复制资源。定义清晰 Input/Output、Version、Validation、Example 和 Upgrade Guide。

避免巨大“企业万能模块”和过度 Wrapper，使底层 Provider 能力不可用。

## State

State 包含资源 ID 和可能的敏感值。使用 Remote Backend、Encryption、Locking、Backup 和严格 RBAC。State Loss 或并发写会产生严重风险。

不要手工编辑 State；使用 `import`、`moved`、`state mv` 等受控操作并备份。

## Provider

锁定 Provider Version，集中 Upgrade 与测试。Provider Alias 支持多 Region/Account，但凭证使用短期 Role/OIDC。

## 环境隔离

生产与非生产使用独立 Account/Subscription、State 和 Credential。Workspace 可用于相似实例，但不应成为弱安全隔离的替代品。

## Pipeline

Format/Validate/Lint/Security → Plan → Review/Policy → Apply → Drift Detect。生产 Apply 使用受保护 Runner 和 Audit。

## 总结

Terraform 的核心风险在 State、权限和大范围 Change。小模块、远程安全 State、版本锁定与 Plan Policy 是基础。
