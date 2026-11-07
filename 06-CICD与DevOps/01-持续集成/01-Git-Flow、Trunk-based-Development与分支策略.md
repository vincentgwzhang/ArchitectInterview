# 01. Git Flow、Trunk-based Development 与分支策略

## Git Flow

使用 `main/develop/feature/release/hotfix` 多类长期分支，适合版本化、低频发布的软件，但 Merge、Cherry-pick 和环境漂移成本较高。

## Trunk-based Development

开发者频繁把小改动合入 `main`，短生命周期 Branch，使用 Feature Flag 隔离未完成能力。优点是减少 Merge Debt、支持 Continuous Delivery；前提是快速 CI、自动测试和良好代码审查。

## 分支策略原则

- `main` 始终可发布。
- Branch 尽量小、短、每日同步。
- Pull Request 有明确 Owner 和 Review SLA。
- 禁止直接修改已发布 Artifact。
- Release 使用不可变 Tag/Commit，而不是复制代码。

## Feature Flag

区分 Release Flag、Experiment Flag、Operational Flag 和 Permission。Flag 有 Owner、到期时间和清理任务；测试开/关组合，避免形成永久条件复杂度。

## Monorepo

Trunk-based 可配合 Path/Dependency-based Impact Detection，只构建受影响项目。共享 Main 不代表所有服务必须同时发布。

## 选择

受严格版本支持的 On-premise Product 可能需要 Release Branch；高频 SaaS 更适合 Trunk-based。也可保留短期 Release Branch，但避免多条长期开发线。

## 指标

Merge Lead Time、Branch Age、Conflict、Deployment Frequency、Change Failure 和 Rework。

## 总结

分支模型应支持交付节奏。现代服务通常用小批量 Trunk-based + Feature Flag，把集成风险持续暴露而非延后集中解决。
