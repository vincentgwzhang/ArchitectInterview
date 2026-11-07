# 04. Prompt 模板、版本管理、缓存与灰度发布

## Prompt Registry

每个 Prompt 记录：

- Use Case 和 Owner。
- Template 与变量 Schema。
- Version/Status。
- Compatible Model。
- Eval Dataset/Result。
- Safety Rule。
- Change History。

生产请求记录不可变 Prompt Version，而不是只记录模板名称。

## 模板

使用结构化 Renderer 和转义，区分 Instruction、User Data、Retrieved Context。变量必须有类型、长度和敏感级别，避免任意字符串拼接导致 Injection 或 Token 失控。

## 缓存

### Exact Cache

相同规范化输入、Prompt、Model 和参数返回已有结果。适合确定性、非敏感、变化少任务。

### Semantic Cache

相似 Query 复用答案，风险更高。必须包含 Tenant、权限、知识版本、Locale 和时间敏感性，设 Similarity Threshold 与 TTL。

### Prefix/Provider Cache

复用稳定 Prompt Prefix，降低 Prefill 成本。模板结构应把稳定 Instruction 放前面，但不能为 Cache 牺牲安全隔离。

## Cache Key

至少考虑：

```text
tenant + user-scope + use-case + prompt-version + model-version
+ normalized input + retrieval-version + policy-version
```

用户私有结果绝不能进入跨用户共享 Cache。

## 灰度发布

1. Offline Eval。
2. Shadow Traffic（不影响用户）。
3. Internal/Dogfood。
4. Small Canary。
5. 按 Tenant/Use Case 扩大。
6. 自动监控并 Rollback。

同时比较 Quality、Safety、Latency、Token、Cost、Tool Success 和 User Outcome。

## Experiment

稳定分桶，避免同一 Conversation 中 Prompt Version 来回切换。高风险场景先 Review，不直接 A/B 给真实用户。

## 总结

Prompt 应像代码和 Model 一样版本化、评测和渐进发布。Cache 必须绑定权限与知识版本，否则性能优化会变成数据泄漏和错误复用。
