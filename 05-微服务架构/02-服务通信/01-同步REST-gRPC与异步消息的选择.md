# 01. 同步 REST/gRPC 与异步消息的选择

## 同步通信

Caller 等待结果，语义直观，适合立即需要响应的 Query/Command。代价是 Temporal Coupling、Latency 累加和 Cascading Failure。

REST 适合公共/Browser/资源 API；gRPC 适合内部强类型、低延迟与 Streaming。二者都需要 Timeout、Deadline、Retry 和 Contract。

## 异步消息

Producer 不等待 Consumer，适合 Domain Event、削峰、Fan-out 和长流程。代价是 Eventual Consistency、重复、乱序、Schema 与排障复杂度。

## 选择问题

- Caller 是否必须立即知道结果？
- Consumer 暂时不可用时是否应继续接受？
- 是 Command 还是已发生 Event？
- 是否需要一对多？
- 允许多长 Staleness？
- 如何处理 Unknown Outcome？

## 常见组合

同步完成核心授权/校验，本地提交后通过 Outbox 发布 Event，通知和 Analytics 异步处理。

## 反模式

- 用 Kafka 实现同步 RPC，双方仍强耦合且更难调试。
- 长同步调用链十几个 Service。
- Event 名为 `UpdateTableRow`，泄露内部模型。
- Fire-and-forget 没有 Durability 和监控。

## 总结

同步优化即时协作，异步优化时间解耦和弹性。根据业务反馈时限与失败语义选择，通常需要混合架构。
