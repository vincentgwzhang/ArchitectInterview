# 02. Workflow、Agent、Planner 与状态机的区别

## Workflow

步骤与分支由开发者预先定义，模型只在局部节点完成分类、生成或抽取。行为可预测、易测试，适合固定业务流程。

```text
Classify -> Retrieve -> Draft -> Validate -> Human Approve
```

## Agent

模型根据目标与当前状态动态选择下一步 Tool 或策略。适合开放式任务，但路径、成本和错误更难预测。

## Planner

先生成任务计划，再由 Executor 执行。计划应视为建议，可在新信息和失败后 Replan。不要让模型生成计划后不经验证直接执行高风险动作。

## 状态机

显式 State、Event 和合法 Transition，适合长期、可恢复流程：

```text
DRAFT -> VALIDATED -> APPROVAL_REQUIRED -> EXECUTING -> COMPLETED
                                     \-> REJECTED
```

LLM 可以建议 Event，但 State Machine 决定是否合法。

## 选择原则

| 场景 | 选择 |
| --- | --- |
| 稳定审批流程 | Workflow/State Machine |
| 开放式资料研究 | Agent |
| 多步骤但可预规划 | Planner + Executor |
| 高风险业务动作 | Deterministic Workflow + Human Approval |

优先选择最简单的可控结构。很多所谓 Agent 实际只需要带 LLM 节点的 Workflow。

## Agent Loop

必须有：

- Max Step/Token/Time Budget。
- Allowed Tool Set。
- State Persistence。
- Duplicate Action Protection。
- Loop Detection。
- Cancellation。
- Human Escalation。

## 可恢复性

长流程应把 State、Tool Call、Result 和 Checkpoint 持久化。Worker Crash 后从安全点恢复，Side Effect 通过 Idempotency 避免重复执行。

## 评估

除了最终成功，还测 Step Count、Tool Error、Unnecessary Action、Cost、Latency、Recovery 和 Policy Violation。

## 总结

Workflow 提供确定性，Agent 提供灵活性，Planner 负责分解，State Machine 保护合法迁移。生产系统通常采用 Hybrid，而不是无限自主 Agent。
