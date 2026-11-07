# 01. Cost Explorer、Budgets、CUR 与成本分摊

## Cost Explorer

交互分析 Service、Account、Region、Tag、Usage Type 和趋势，适合日常调查和 Forecast。先理解 Amortized、Unblended、Net Cost 等口径。

## AWS Budgets

按 Cost、Usage、Reservation/Savings Plans 设置阈值和通知/Action。Budget 是事后/趋势 Guardrail，不是硬实时断路器。

## CUR

Cost and Usage Report 提供最细账单明细，可存 S3 并用 Athena/Redshift/BI 分析。建立稳定数据模型、Account/Tag 映射和访问控制。

## 分摊

优先直接归属：Account、Resource Tag、Service。共享 Network/Platform/Support 使用透明 Driver 分摊，如 Usage、Team 或收入。区分 Showback 与 Chargeback。

## Tag Governance

激活 Cost Allocation Tag，自动强制 Owner/Cost Center，并处理不可 Tag 资源。未分配成本本身应成为 KPI。

## Dashboard

总成本之外看 Unit Cost、Forecast Variance、Anomaly、Commitment Coverage 和 Shared Cost。

## 总结

FinOps 从准确可解释的成本数据开始。Explorer 用于探索，Budgets 用于 Guardrail，CUR 支撑精细分摊和单位经济分析。
