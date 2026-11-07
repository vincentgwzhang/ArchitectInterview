# 02. Offline Evaluation、Online Evaluation 与 A/B Test

## Offline Evaluation

在发布前对固定 Dataset 运行，快速、可重复、适合 Regression。

指标：Task Success、Correctness、Groundedness、Tool Accuracy、Safety、Latency 和 Cost。Human Review 是高质量基准；规则和 LLM Judge 用于扩展，但需校准。

局限：Dataset 可能不代表真实分布，无法完全测用户行为和系统环境。

## Online Evaluation

生产中观察：

- User Feedback。
- Task Completion。
- Retry/Rephrase。
- Escalation。
- Citation Click。
- Tool Outcome。
- Safety Incident。
- Latency/Cost。

需要 Sampling、Privacy、Human Review 和 Drift Detection。

## A/B Test

随机稳定分桶比较两个 Version。提前定义 Primary Metric、Guardrail Metric、样本量和停止规则。

AI 输出方差较大，实验期间固定或记录 Model Version。按 Conversation 而非单次 Message 分桶，避免用户体验不一致。

## Shadow Test

新版本接收生产输入但不影响用户，适合比较质量、Tool Plan 和成本。注意不要执行 Side Effect Tool，并遵守数据用途与 Provider Policy。

## Interleaving/Pairwise

对 Search/RAG 可由 Reviewer 盲评两个答案，减少绝对打分差异。随机左右顺序防止 Position Bias。

## 发布判断

不能只看平均质量上涨：检查关键 Category、安全、少数语言、P95 Latency、Cost 和失败长尾。高风险 Regression 可否决整体小幅提升。

## Feedback Loop 风险

生产 Feedback 有 Selection Bias；自动把高评分内容作为训练数据可能放大偏见或被 Data Poisoning。需要清洗、审核和 Lineage。

## 总结

Offline Eval 提供可重复门禁，Online Eval 验证真实价值，A/B Test 建立因果证据。三者互补，并需同时关注质量、安全、性能与成本。
