# 05. RAG 评估：召回率、相关性、忠实度与答案质量

## 分层评估

RAG 需要分开测：

1. Ingestion：内容是否正确进入 Index？
2. Retrieval：相关证据是否进入 Top K？
3. Context：选择的内容是否相关、无冲突？
4. Generation：答案是否使用证据并满足任务？
5. End-to-end：用户能否完成目标？

否则一个总分无法定位改进方向。

## Dataset

来源包括真实匿名 Query、Support Ticket、Domain Expert 编写和失败案例。每条样本可包含：

- Query。
- Relevant Document/Chunk。
- Reference Answer 或 Rubric。
- Required Citation。
- User/Tenant/ACL Context。
- Difficulty/Category。

划分固定 Regression Set 与持续更新的 Challenge Set，防止只优化已知题目。

## Retrieval Metric

- Recall@K。
- Precision@K。
- MRR。
- nDCG。
- ACL Leakage Rate（必须为零）。
- Latency 与 Zero-result Rate。

## Generation Metric

- Faithfulness/Groundedness：Claim 是否由 Context 支持。
- Answer Relevance。
- Correctness。
- Citation Correctness/Completeness。
- Completeness。
- Refusal Correctness。
- Style/Format Compliance。

## LLM-as-Judge

可扩大评测规模，但存在 Bias、Position Effect 和自身错误。应：

- 使用明确 Rubric。
- 隐藏 Model Identity。
- 随机答案顺序。
- 用 Human-labeled Set 校准。
- 高风险结论由专家复核。

## Online Evaluation

观察 User Feedback、Rephrase、Citation Click、Task Completion、Escalation 和 Abandonment。Thumbs-up/down 信号稀疏且有偏，不能单独代表质量。

## Experiment

比较 Chunk、Embedding、Retriever、Reranker、Prompt 和 Model 时一次控制少量变量，保存完整 Pipeline Version。除了质量，还报告 P95 Latency 与 Cost。

## 发布门禁

- 核心 Dataset 不退化。
- Safety/ACL 测试通过。
- 关键 Category 达到阈值。
- Latency/Cost 在 Budget 内。
- Canary 后 Online Signal 正常。

## 总结

RAG Evaluation 必须可诊断、可重现并覆盖安全。分层 Metric、代表性 Dataset、人机结合评审和线上反馈构成持续改进闭环。
