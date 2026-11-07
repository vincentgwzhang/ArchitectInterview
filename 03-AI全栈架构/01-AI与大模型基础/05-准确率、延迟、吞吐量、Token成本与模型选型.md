# 05. 准确率、延迟、吞吐量、Token 成本与模型选型

## 多目标选择

Model Selection 同时优化：

- Quality/Task Success。
- Factuality 与 Safety。
- Time to First Token（TTFT）。
- Time per Output Token。
- Throughput 与 Rate Limit。
- Input/Output Token Cost。
- Context Window。
- Tool/Structured Output 能力。
- Data Residency 与 Privacy。

不存在所有维度都最优的 Model。

## Latency 分解

```text
End-to-end = Gateway + Retrieval + Rerank
           + Model queue + Prefill + Decode
           + Tool calls + Post-processing
```

TTFT 主要受 Queue、Prompt 长度和 Prefill 影响；完整时间还受 Output Token 数影响。Streaming 改善感知等待，但不一定减少总计算时间。

## Cost Model

```text
Request cost = input tokens × input price
             + output tokens × output price
             + retrieval/rerank/tool cost
```

还包括 Embedding、Vector Storage、Observability、Retry 和 Engineering TCO。

## Benchmark

使用自己的 Dataset，而不是只看通用榜单：

- 按 Use Case 和风险分组。
- Blind Review 或明确 Rubric。
- 测 P50/P95 Latency。
- 记录 Token、Error、Tool Success。
- 多次运行评估 Variance。
- 包含恶意和边界输入。

## Routing

常见 Cascade：

1. 规则或小 Model 处理简单任务。
2. 中型 Model 处理多数请求。
3. 低 Confidence 或复杂任务升级到强 Model。
4. 高风险结果进入 Human Review。

Router 本身也需评估，错误降级会损害质量，过度升级则无法节省成本。

## 优化手段

- 删除重复 Prompt 与无关 Context。
- 限制 Output Length。
- Cache 确定性或语义等价请求。
- Batch Embedding/Offline Job。
- 选择更小 Model。
- 并行独立 Retrieval/Tool。
- 对高成本用户设置 Quota。

## 决策记录

记录 Model Version、Eval Dataset、Metric、价格假设、Fallback 和 Revisit Trigger。供应商更新 Model 后应重新跑 Regression Eval。

## 总结

Model Selection 是可测量的架构决策。用真实任务评估质量，以端到端方式测 Latency 和 Cost，并通过 Routing 与 Guardrail 达到业务目标。
