# 04. 云迁移策略：Rehost、Replatform 与 Refactor

## 常见策略

- Rehost：Lift-and-shift，快但保留旧问题。
- Replatform：少量修改使用 Managed DB/Container 等。
- Refactor：重构架构获得 Cloud-native 能力，成本风险高。
- Repurchase：改用 SaaS。
- Retire：关闭。
- Retain：暂不迁移。

## 选择

按业务价值、寿命、合规、耦合、性能、License、技能和 Deadline，不对所有应用使用同一策略。

## Landing Zone

迁移前建立 Account、Identity、Network、Logging、Security、Backup、Cost 和 IaC。否则只是把 Data Center 混乱搬到 Cloud。

## Wave

先低风险 Representative Workload 验证 Factory，再按 Dependency Group 迁移。每 Wave 有 Entry/Exit、Cutover、Rollback 和 Hypercare。

## Modernize Later 风险

Rehost 可加快退出机房，但必须记录 Modernization Trigger，否则高成本、脆弱架构会永久存在。

## 总结

云迁移是 Portfolio 决策。不同系统选择不同“R”，并通过 Landing Zone、Wave 和业务指标控制风险。
