# 04. Step Functions 与分布式工作流编排

## 工作流

Step Functions 用 State Machine 编排 Lambda、ECS、AWS SDK 和人工 Callback，持久化状态并管理 Retry/Timeout/Parallel/Choice。

- Standard：长运行、可审计、Exactly-once Workflow Execution 语义更适合关键流程。
- Express：高吞吐短流程，计费和执行语义不同。

## 设计

每个 Task 有 Timeout、Retry（仅 Transient）、Catch 和 Result Path。避免把巨大 Payload 放 State，使用 S3 Pointer。

## Saga

用显式 State 调用 Reserve、Pay、Confirm，失败进入 Compensation。每个 Activity 仍需 Idempotent，因为 Retry/Timeout 可能导致重复 Side Effect。

## Callback

Task Token 支持 Human Approval 或外部长任务。Token 是 Secret，安全传递并设置 Timeout。

## Versioning

正在运行的 Workflow 可能持续数天。修改 Definition 时考虑旧 Execution，使用 Version/Alias 或兼容 Worker。

## Observability

Execution History、CloudWatch Metric/Log、Trace 和业务 ID。大量 History/Payload 需注意成本和敏感数据。

## 总结

Step Functions 把长流程、Retry 和状态持久化平台化，但业务幂等、补偿和版本仍由应用设计。
