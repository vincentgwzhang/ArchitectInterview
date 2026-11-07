# 02. Java API、AI Gateway、RAG 与向量检索

## 请求链

```text
React -> Java BFF/API -> AI Gateway
-> Query Rewrite -> Hybrid Search + ACL
-> Rerank -> Model -> Citation Validate -> Stream
```

Java 层负责 Session、Tenant、业务权限、Quota 和统一 SSE；AI Gateway 负责 Model Routing、Prompt Version、Token/Cost、Safety 和 Provider Adapter。

## Retrieval

BM25 + Vector 召回，使用 RRF 融合后 Rerank。Metadata 包含 Tenant、ACL、Language、Effective Date、Source Version。ACL 必须在检索前执行。

## Context

Dedup/Parent Selection/Compression，分配 Token Budget。Prompt 只允许根据证据回答，不足时 Abstain。Backend 验证 Citation ID 属于本次 Retrieval Set。

## Reliability

Timeout/Circuit/Rate Limit；Vector 失败可回退 Keyword Search；模型失败可返回相关文档，而不是无依据答案。取消传播到 Model。

## 总结

Java 提供企业业务边界，AI Gateway 隔离模型变化，RAG 用可授权证据提供事实；每层职责不可混淆。
