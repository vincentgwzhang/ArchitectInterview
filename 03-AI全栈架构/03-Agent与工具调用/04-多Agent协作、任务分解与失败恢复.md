# 04. 多 Agent 协作、任务分解与失败恢复

## 何时需要 Multi-agent

适合任务可明确分解、需要不同 Tool/权限/专业上下文，或可并行探索。若一个 Agent 加几个 Tool 已能完成，Multi-agent 只会增加 Latency、Cost 和 Debug 难度。

## 常见模式

- Supervisor：分配任务并合并结果。
- Planner/Executor：计划与执行分离。
- Specialist Team：Research、Code、Review 等角色。
- Debate/Judge：多个候选再评审。
- Map-Reduce：并行处理多个文档后汇总。

## Task Contract

子任务应包含：

- Objective。
- Inputs 和允许访问的 Context。
- Allowed Tools/Permission。
- Output Schema。
- Budget/Deadline。
- Success Criteria。
- Failure/Escalation 方式。

自由文本“帮我研究一下”难以合并和验证。

## 并行与依赖

构建 DAG：无依赖任务并行，有依赖任务等待明确 Artifact。限制 Fan-out，避免 Token 与 Tool Call 指数增长。

## 结果合并

Supervisor 不能只拼接答案。需要：

- 去重。
- 冲突检测。
- Source/Citation 保留。
- Confidence/Uncertainty。
- Schema Validation。
- 必要时调用 Verifier。

## 失败恢复

- 每个 Task 持久化状态。
- Tool Call Idempotency。
- 有界 Retry + Backoff。
- 超时和 Cancellation Propagation。
- Partial Result。
- Human Escalation。
- Resume from Checkpoint。

一个 Agent 失败不应导致整个 Workflow 从头执行。

## 安全

按 Agent 最小权限分配 Tool。不要让 Research Agent 获得写生产数据权限。Agent 间 Message 视为不可信输入，防止 Prompt Injection 跨层传播。

## 评估

比较 Single-agent Baseline：Task Success 是否显著提升？同时测 Step、Cost、Latency、冲突、失败恢复和人工介入。没有收益就回归简单架构。

## 总结

Multi-agent 的价值来自结构化分工和隔离，而不是模拟组织称谓。明确 Task Contract、DAG、Budget、权限和恢复机制是成功条件。
