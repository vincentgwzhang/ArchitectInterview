# 03. 幻觉检测、引用验证、Guardrails 与输出校验

## 幻觉类型

- 无证据生成事实。
- 错读或合并冲突证据。
- 引用真实来源但来源不支持 Claim。
- Tool Result 被错误解释。
- 过时知识被当成当前事实。

没有一个通用“Hallucination Detector”能保证发现所有错误，应采用分层控制。

## Grounding

- 为事实回答提供受权 Retrieval/Tool。
- Prompt 要求只使用给定证据。
- 证据不足时允许 Abstain。
- 将事实 Claim 与 Citation 绑定。
- 高风险字段使用确定性 Source 直接生成。

## Citation Validation

Backend 验证：

1. Citation ID 属于本次 Retrieval Set。
2. 用户有权访问 Source。
3. Source Version 有效。
4. 引用片段包含支持 Claim 的文本。

最后一步可由规则、NLI/LLM Judge 和 Human Sampling 结合，但不能把“有引用”直接当成“引用正确”。

## Guardrails

层次包括：

- Input Schema/Policy。
- Retrieval ACL。
- Tool Allowlist 与 Authorization。
- Output JSON Schema。
- Business Rule Engine。
- Content/Safety Filter。
- Human Approval。

确定性规则优先处理金额、权限、状态迁移等明确约束；LLM Guardrail 适合语义分类，但仍有概率错误。

## Output Validation

结构验证后继续检查：

- Range、Enum、Cross-field Constraint。
- Identifier 是否存在。
- 数值重新计算。
- URL Domain Allowlist。
- SQL/Code 不自动执行。
- PII/Secret 泄露。

失败时可 Repair 一次、降级模板、拒绝或升级人工；避免无限自我修复 Loop。

## 评估

测 Groundedness、Citation Correctness、Abstention Precision/Recall、Guardrail Bypass、False Block 和高风险 Field Error。

## 总结

降低幻觉依赖可靠证据、允许拒答、引用验证和确定性业务校验。Guardrail 是多层架构，不是一个“安全 Prompt”。
