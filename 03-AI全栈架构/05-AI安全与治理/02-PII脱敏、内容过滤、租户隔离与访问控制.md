# 02. PII 脱敏、内容过滤、租户隔离与访问控制

## 数据分类

先识别 PII、Secret、Payment、Health、Confidential IP 和 Public Data。不同类别决定可使用的 Model、Region、Retention、Logging 和 Human Review。

## PII 处理

流程：

```text
Detect -> Classify -> Redact/Tokenize -> Process -> Controlled Rehydrate
```

方法包括 Regex、NER、规则和专用 DLP Service。Regex 适合确定格式，不能覆盖所有姓名、地址和上下文 PII。

Tokenization 允许在授权边界内恢复原值；不可逆 Masking 适合 Analytics/Eval。保持原值与映射表分离并严格审计。

## 内容过滤

可以在 Input、Retrieved Context、Model Output 和 Tool Action 多层执行。策略应按 Use Case 和用户年龄/地区配置，并区分：

- Block。
- Safe Completion。
- Warn。
- Human Review。
- Allow with Audit。

过滤器有 False Positive/Negative，需要 Appeal、Evaluation 和版本管理。

## Multi-tenancy

租户隔离要贯穿：

- Identity Token。
- Application Query。
- Database Row/Schema。
- Vector Namespace/Metadata Filter。
- Cache Key。
- Object Storage Prefix/Policy。
- Log/Analytics Access。
- Encryption Key（按风险选择）。

Tenant ID 不应由 Client 自由指定；从可信身份上下文派生。

## Authorization

RAG 必须在检索时执行 Document/Chunk ACL。先跨权限检索再让模型“不要使用”已泄露内容是不安全的。

Tool Call 同样需要 Resource-level Authorization。Service Account 的权限不能替代 End-user Delegation 或 Policy Check。

## Model Provider

评估数据是否用于训练、Retention、Region、Subprocessor、Encryption、Zero-retention 选项和 Contract。敏感数据可能要求专用 Deployment 或自托管。

## 测试

- Cross-tenant Retrieval。
- Cache Poison/Leak。
- Deleted User Data。
- Log Redaction。
- Filter Bypass。
- Support/Admin Access。

## 总结

AI 数据安全是端到端 Data Flow 问题。脱敏、内容策略、可信 Tenant Context、检索前 ACL 和受控 Provider 共同构成隔离。
