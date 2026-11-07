# 03. SSE、WebSocket、异步任务与长耗时工作流

## SSE

Server-Sent Events 是 Server → Client 单向 Event Stream，基于 HTTP，适合 LLM Token Streaming、进度和通知。

优点：简单、自动重连语义、容易通过 Web Infrastructure。限制：Client → Server 仍需普通请求，部分 Proxy 有 Buffer/Timeout，需要 Heartbeat 和正确 Header。

## WebSocket

全双工长连接，适合实时协作、语音和频繁双向 Event。代价是 Connection State、Scaling、Load Balancer、Heartbeat、Backpressure 和断线恢复更复杂。

聊天文字生成通常 SSE 已足够；不要为了“实时”默认 WebSocket。

## 异步任务

超过 HTTP Deadline 的 Job 使用：

```text
POST /jobs -> 202 + jobId
Worker consumes queue
GET /jobs/{id} or SSE progress
```

Job State：Queued、Running、WaitingApproval、Succeeded、Failed、Cancelled。保存 Checkpoint、Progress、Artifact 和 Error。

## 长工作流

使用 Durable Workflow/State Machine 管理：

- Retry 和 Backoff。
- Timer。
- Human Approval。
- Compensation。
- Crash Recovery。
- Versioning。

不要把数小时 Agent Loop 只保存在单个 Process Memory。

## 幂等与取消

提交使用 Idempotency Key；Worker Side Effect 也需幂等。取消是协作式：标记 Cancellation，Worker 在安全 Checkpoint 停止，并传播到 Model/Tool。

## Backpressure

限制每 Tenant 并发、Queue Depth 和 Output Buffer。慢 Client 不能无限占用内存；必要时持久化 Event 并允许 Cursor Resume。

## 选择

- 单向 Token：SSE/Fetch Stream。
- 高频双向媒体：WebSocket/WebRTC。
- 分钟级任务：Async Job + Queue。
- 小时/天级且需恢复：Durable Workflow。

## 总结

通信协议应匹配方向、持续时间和恢复要求。长耗时 AI 任务需要持久状态、幂等、进度、取消和资源配额，而不只是保持 HTTP 连接。
