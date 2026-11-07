# 03. Lambda、API Gateway、EventBridge 与 Serverless

## Lambda

事件驱动 Function，自动 Scale，按调用/执行计费。适合短时、无状态、事件与 API。限制包括 Timeout、Concurrency、Cold Start、Package、`/tmp` 和下游连接。

设置 Reserved Concurrency 保护下游，Provisioned Concurrency 降低关键路径 Cold Start。代码必须 Idempotent。

## API Gateway

提供 REST/HTTP/WebSocket API、Auth、Throttle、Stage 和 Integration。HTTP API 较简洁低成本，REST API 功能更丰富。业务 Authorization 仍由 Backend 验证。

## EventBridge

Event Bus 按 Pattern 路由 AWS/SaaS/自定义 Event，适合松耦合集成。定义 Schema、Archive/Replay、Retry 和 DLQ。不是高吞吐 Streaming 的通用替代品。

## Serverless 设计

- Event Source Mapping 和 Batch Failure。
- At-least-once + Idempotency。
- DLQ/Destination。
- Step Functions 管长流程。
- X-Ray/CloudWatch/OTel Observability。
- IAM 每 Function 最小权限。

## 成本

低/波动负载经济，高持续计算、长时间或大量数据传输可能更贵。评估端到端服务费用，而非只看 Lambda。

## 总结

Serverless 减少服务器管理，但把架构重点转向事件、配额、幂等、权限和可观测性。
