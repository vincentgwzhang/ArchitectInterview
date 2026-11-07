# 02. Embedding 模型、向量数据库与索引策略

## Embedding Model 选择

评估：

- Domain 和 Language。
- Dimension。
- 最大输入 Token。
- Query/Document 是否使用不同前缀。
- Retrieval Benchmark。
- Latency、Cost、Data Policy。

不要只看通用 Leaderboard。用自己的 Query-Document Relevance Dataset 测 Recall@K、MRR 或 nDCG。

## 向量相似度

- Cosine Similarity：比较方向。
- Dot Product：常与归一化向量结合。
- Euclidean Distance：比较距离。

Embedding Model 建议使用的 Metric 必须与 Index 配置一致。

## ANN Index

Exact Search 在大规模下昂贵，常用 Approximate Nearest Neighbor：

- HNSW：查询快、Recall 高，内存和构建成本较高。
- IVF：先聚类再搜索部分分区，需调参和训练。
- Product Quantization：压缩向量，降低存储但损失精度。

调参在 Recall、Latency、Memory 和 Build Time 之间权衡。

## Vector Database 选型

考虑：

- 数据规模与 QPS。
- Metadata Filter 能力和性能。
- Update/Delete 语义。
- Multi-tenancy 与 ACL。
- Backup、Replication、DR。
- Hybrid Search。
- Observability。
- Managed vs Self-hosted TCO。

PostgreSQL Extension 可能足够中小规模；专用 Vector DB 适合更大规模或复杂检索，但增加平台成本。

## Index Strategy

- 按 Tenant/Region 分 Index 还是共享 + Filter。
- Namespace 是否造成大量小分区。
- Hot/Warm 数据分层。
- Embedding Version 并行 Index。
- Blue/Green Reindex。

切换 Model 时不能混用不同向量空间。建立新 Index，完成 Backfill/Eval 后原子切流。

## 安全

Metadata ACL Filter 必须在 Retrieval 执行，而不是检索后再让 LLM 忽略。验证 Filter 不会因 ANN、Cache 或 Namespace 配置被绕过。

## 总结

Embedding 与 Index 是一套联合设计。以真实 Retrieval Eval 选择 Model，用规模、过滤、安全、更新和运营要求选择 Store，并为 Reindex 留出演进路径。
