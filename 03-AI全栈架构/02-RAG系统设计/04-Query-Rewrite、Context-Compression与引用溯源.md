# 04. Query Rewrite、Context Compression 与引用溯源

## Query Rewrite

用户问题可能依赖对话上下文：

> “它支持多区域吗？”

Retriever 不知道“它”是什么。可以生成 Standalone Query：

> “Product X 是否支持 Multi-region Deployment？”

Rewrite 类型包括：

- Coreference Resolution。
- 缩写扩展。
- Multi-query Expansion。
- HyDE。
- Query Decomposition。

必须保留原始 Query，避免 Rewrite 改变用户意图。对 Product ID 等精确 Token 不应随意改写。

## Query Decomposition

复杂问题拆成子问题并独立检索，再合并证据。例如“比较 A/B 的价格、安全和 SLA”可拆成三个维度。需要控制并行请求数、重复证据和合并错误。

## Context Compression

目标是在不丢失关键证据的前提下降低 Token：

- 提取与 Query 相关句子。
- Parent Document 中选择局部 Section。
- 去重相邻 Chunk。
- 摘要长 Tool Result。
- 按 Evidence Score 截断。

Compression Model 可能误删条件或否定词。高风险内容应保留原文片段并做 Eval。

## 引用溯源

每个 Chunk 必须带 Source ID、Title、URL、Page/Section、Version 和 Character Offset。Model 输出 Citation 时，Backend 应验证 Citation ID 确实来自本次检索集合。

Citation Quality 包括：

- Correctness：来源是否支持 Claim。
- Completeness：重要 Claim 是否有引用。
- Freshness：是否引用有效版本。
- Accessibility：用户是否有权打开来源。

## UI

引用应可点击并展示原文 Highlight。若用户无文档访问权限，不应仅隐藏链接却展示内容；权限必须在 Retrieval 前执行。

## 失败处理

证据不足时明确回答无法确认，并给出已搜索范围。不要通过更强 Prompt 强迫模型提供答案。

## 总结

Rewrite 改善意图表达，Compression 控制 Context，Citation 建立可验证信任。三者都必须保留 Lineage，并通过独立评测防止改变事实。
