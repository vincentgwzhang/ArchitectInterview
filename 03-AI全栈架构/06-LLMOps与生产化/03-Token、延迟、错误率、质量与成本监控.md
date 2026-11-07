# 03. Token、延迟、错误率、质量与成本监控

## 指标层次

### Infrastructure

CPU/GPU、Memory、Queue、Rate Limit、Connection。

### Model/API

Request、TTFT、Tokens/sec、Input/Output Token、Timeout、Provider Error。

### Pipeline

Retrieval/Rerank Latency、No-hit、Tool Success、Agent Step、Cache Hit。

### Product/Quality

Task Success、Groundedness Sample、Citation、User Feedback、Escalation、Safety Incident。

### Business/Cost

Cost per Request、per Successful Task、per Tenant、per Feature。

## Latency

分解 Gateway、Queue、Retrieval、Model Prefill/Decode、Tool 和 Post-process。只看 End-to-end 无法定位瓶颈；只看 Model 又会遗漏实际 UX。

## Token 与 Cost

监控 Token Distribution 而非只看平均值。异常长 Prompt 可能来自对话失控、重复 Context 或攻击。设置 Per-request、User/Tenant 和 Daily Budget。

## Error Taxonomy

- Client Validation。
- Policy Refusal。
- Provider Rate Limit。
- Timeout/Network。
- Tool Failure。
- Schema Validation。
- Retrieval Empty。
- Unknown Outcome。

不要把正常 Safety Refusal 混入系统错误率。

## Quality Monitoring

质量无法像 CPU 一样完全实时准确测量。结合：

- 自动 Proxy Metric。
- 抽样 LLM Judge。
- Human Review。
- User Outcome。
- Canary Dataset Replay。

所有质量分数关联 Model、Prompt、Retriever 和 Judge Version。

## Alert

对 SLO Burn Rate、Provider Error、Tool Failure、Safety Spike、Cost Anomaly 和质量显著下降告警。每个 Alert 有 Owner、Runbook 和可执行动作。

## Dashboard

按 Use Case、Tenant Tier、Language、Release Version 和 Provider 分段，但避免在 Metric Label 放 User ID 等高 Cardinality 数据。

## 总结

LLM Monitoring 需要把技术、质量和业务连接起来。Unit Cost 和成功任务比单纯 Token 总量更能指导优化。
