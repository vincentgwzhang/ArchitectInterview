# 05. 单位经济成本、成本异常检测与 FinOps 文化

## Unit Economics

总账单随业务增长上升不一定是坏事。计算：

- Cost per Order。
- Cost per Active User。
- Cost per 1,000 API Calls。
- Cost per AI Answer。
- Gross Margin by Tenant/Product。

Unit 定义必须稳定并与业务量关联。

## 异常检测

使用 Cost Anomaly Detection/Budget/CUR 发现突增，按 Account、Service、Usage Type 和 Tag 路由 Owner。设置百分比与绝对金额双阈值，避免小成本噪音。

## 反馈

团队看到近实时成本、Forecast 和 Recommendation；Architecture Review 包含 Cost Estimate；Incident/Postmortem 也分析成本异常。

## FinOps 文化

Finance、Engineering、Product 共同优化业务价值，而不是中央团队单纯削减。Team 对可控成本负责，Platform 提供数据、工具和 Golden Path。

## Optimization Backlog

每项包含预计 Savings、Engineering Cost、Risk、Owner 和验证。优先高价值低风险，完成后用账单验证真实节省，避免“纸面 Savings”。

## Guardrail

保护 SLO、Security、Compliance 和员工效率。过度节省造成 Incident 的业务成本可能更高。

## 总结

FinOps 是持续决策机制。Unit Cost 连接云支出和业务价值，异常检测缩短反馈，团队 Ownership 让优化可持续。
