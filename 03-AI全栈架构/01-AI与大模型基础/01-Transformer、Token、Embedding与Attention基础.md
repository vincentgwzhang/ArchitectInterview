# 01. Transformer、Token、Embedding 与 Attention 基础

## Token

LLM 处理的基本单位不是“单词”，而是 Token。Tokenizer 会把文本拆成 Subword、字符或符号 ID。不同语言和代码的 Token Efficiency 不同，因此字符数不能直接等于 Token 数。

Token 影响：

- Context Window。
- Latency 与 Throughput。
- API Cost。
- Chunk Size。
- Output 上限。

## Embedding

Embedding 把 Token、句子或文档映射为高维向量。语义相近内容通常在向量空间距离更近。

常见相似度：Cosine Similarity、Dot Product、Euclidean Distance。必须根据 Embedding Model 和 Vector Database 配置选择。

Embedding 用于 Retrieval、Clustering、Recommendation 和 Semantic Cache，但它不自动理解权限、时间和精确数值，需要 Metadata Filter 与其他检索方式配合。

## Attention

Self-attention 让每个 Token 根据 Query、Key、Value 关注序列中的其他 Token。简化表达：

```text
Attention(Q,K,V) = softmax(QKᵀ / √d) V
```

Multi-head Attention 在不同表示子空间学习不同关系。Position Information 让模型感知顺序。

## Transformer

Transformer 通过多层 Attention、Feed-forward Network、Residual Connection 和 Normalization 构建表示。现代生成式 LLM 多使用 Decoder-style 自回归预测下一个 Token。

Inference 时模型反复生成下一个 Token；KV Cache 保存已有 Token 的中间结果，降低重复计算，但消耗显存。

## 架构影响

- 长 Prompt 增加 Prefill Latency 和成本。
- 长 Output 增加逐 Token Decode 时间。
- Context Window 大不代表模型能同等有效使用所有内容。
- Embedding Model 或 Chunk 变化需要重新 Index 与评测。
- Exact Match、日期、ID 等信息常需 Hybrid Search。

## 面试要点

不需要推导所有数学细节，但应能把 Token、Context、Attention、Embedding 与 RAG 的性能、成本和质量联系起来。

## 自测题

1. 为什么中文字符数不能直接用于估算 API Token？
2. KV Cache 解决什么问题，又带来什么成本？
3. Embedding 相似为什么不能替代 Authorization Filter？

## 总结

Token 是计算与计费单位，Embedding 提供语义空间，Attention 建模上下文关系，Transformer 将它们组合成可扩展的序列模型。
