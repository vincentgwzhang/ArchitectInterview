# 01. Prompt、模型、数据集和评测集的版本管理

## 可重现单元

一次 AI Release 不是单个 Model Version，而是组合：

```text
Application Code
+ Prompt Version
+ Model/Parameter
+ Retrieval Pipeline/Index
+ Tool Schema
+ Policy/Guardrail
+ Eval Dataset
```

任何一项变化都可能改变输出。

## Version Artifact

- Prompt：不可变模板、变量 Schema、Owner。
- Model：Provider、具体版本/Endpoint、参数。
- Dataset：Snapshot、Schema、Lineage、License。
- Eval：Dataset Version、Rubric、Judge Version、Result。
- Index：Source Snapshot、Chunker、Embedding、Build ID。

使用唯一 Release Manifest 关联这些 Artifact。

## Dataset 管理

Dataset 应有：

- Data Card。
- 来源和 Consent。
- PII 处理。
- Train/Validation/Test Split。
- 去重与 Leakage Check。
- Label Guideline 和 Agreement。
- Retention/Deletion。

不要在调 Prompt 时反复查看并优化 Test Set，避免 Overfitting。

## Evaluation Set

分为：

- Golden Set：稳定回归。
- Challenge Set：困难/攻击案例。
- Fresh Production Sample：检测分布变化。
- Safety Set：高风险和拒答。

修改 Rubric 或 LLM Judge 也要版本化，否则分数不可比较。

## Promotion

```text
Experiment -> Offline Eval -> Review -> Staging
-> Canary -> Production -> Monitor
```

Artifact 在环境间 Promote，不应在生产临时编辑 Prompt 而没有版本记录。

## Rollback

保留上一个完整 Release Manifest。只回滚 Prompt 但不回滚不兼容 Tool Schema，可能无法恢复。

## 总结

LLMOps 的基础是完整组合可重现。通过 Release Manifest、Data Lineage、固定 Eval 和不可变 Promotion，才能可靠比较和回滚。
