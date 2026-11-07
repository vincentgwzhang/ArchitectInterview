# 04. Fine-tuning、Prompt Engineering 与 RAG 的选择

## 三种手段解决不同问题

### Prompt Engineering

用于明确任务、格式、规则和示例。迭代快、成本低，应作为第一步。

### RAG

用于提供外部、私有、经常更新且可引用的知识。数据保留在检索系统，更新后不必重新训练 Model。

### Fine-tuning

用于稳定行为、风格、格式或特定任务模式。它不是可靠注入大量实时事实的首选方式。

## 决策表

| 问题 | 优先方案 |
| --- | --- |
| 输出格式不稳定 | Structured Output / Prompt，必要时 Fine-tuning |
| 需要最新内部文档 | RAG |
| 特定品牌语气 | Prompt 或 Fine-tuning |
| 大量高质量输入输出样本 | 可评估 Fine-tuning |
| 复杂实时计算 | Tool Calling |
| 权限相关知识 | RAG + Metadata ACL Filter |

这些方案可以组合：Fine-tuned Model 负责行为，RAG 提供事实，Tool 提供实时状态。

## Fine-tuning 流程

1. 定义 Baseline 和 Success Metric。
2. 收集代表性高质量 Dataset。
3. 清理 PII、错误标签和重复样本。
4. 划分 Train/Validation/Test。
5. 训练并对比 Baseline。
6. 做 Safety/Regression Eval。
7. Version、Canary、Monitor 和 Rollback。

训练数据质量通常比数量更重要。必须有 Data License、Consent 和删除机制。

## RAG 的代价

RAG 引入 Ingestion、Chunk、Embedding、Vector Store、Retrieval、Rerank、Citation 与权限控制。失败可能来自 Retrieval，不一定是 Model。

## 常见错误

- 用 Fine-tuning 记忆频繁变化的产品价格。
- RAG 没有评估 Retrieval Recall。
- 认为 Prompt 能替代 Backend Authorization。
- 在没有 Baseline Eval 时进行昂贵训练。
- 把 Tool Calling 与 RAG 混淆：Tool 执行动作/获取实时结构化数据，RAG 检索知识。

## 总结

先用 Prompt 和结构化输出建立 Baseline；需要外部知识时用 RAG；需要稳定学习行为且有高质量数据时考虑 Fine-tuning；实时事实和动作交给 Tool。
