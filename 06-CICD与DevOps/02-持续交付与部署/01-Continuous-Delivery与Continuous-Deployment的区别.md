# 01. Continuous Delivery 与 Continuous Deployment 的区别

## 定义

- Continuous Delivery：每次变更都保持可部署，生产发布可由人工业务决定。
- Continuous Deployment：所有通过自动门禁的变更自动进入生产。

两者都依赖 Continuous Integration、自动测试、不可变 Artifact、环境一致性和快速恢复。

## Deployment 与 Release

Deployment 是代码进入环境；Release 是用户获得 Feature。Feature Flag 可以先部署后发布，降低技术部署与业务时点耦合。

## Pipeline

```text
Commit -> Validate -> Build/Sign -> Test -> Publish
-> Deploy Staging -> Verify -> Promote -> Production -> Observe
```

同一 Artifact Promote，生产不重新编译。

## 人工审批

人工审批适合风险/合规决策，不应补偿自动测试不足。审批界面展示 Change、Evidence、Risk、Artifact 和 Rollback，设置 SLA 避免排队。

## 自动部署条件

- 小批量 Trunk-based。
- 强自动 Test/Scan。
- Progressive Delivery。
- SLO/Release Metric。
- 自动 Rollback/Kill Switch。
- 团队 Build-to-Run Ownership。

## 衡量

DORA：Deployment Frequency、Lead Time、Change Failure Rate、Time to Restore。不能只追求频率而牺牲稳定性。

## 总结

Delivery 保证“随时可以发”，Deployment 自动决定“通过就发”。组织应按风险选择，但基础工程能力相同。
