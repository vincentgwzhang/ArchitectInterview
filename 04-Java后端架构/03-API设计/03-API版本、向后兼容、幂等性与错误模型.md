# 03. API 版本、向后兼容、幂等性与错误模型

## 向后兼容

优先演进而非频繁 `/v2`：

- 增加 Optional Field。
- 不删除/重命名已有 Field。
- 不改变字段语义与单位。
- Consumer 对未知 Enum 容错。
- 新字段提供合理 Default。

用 Consumer-driven Contract、Schema Compatibility 和流量观察验证。

## Version Strategy

URI、Header 或 Media Type 都可版本化。选择一致比形式更重要。只有 Breaking Change 才创建新 Major Version，并提供 Deprecation Date、Migration Guide 与 Usage Dashboard。

## Idempotency

Payment、Order Create 等 POST 使用 Idempotency Key。Server 保存 Key、Request Fingerprint、状态和 Response：

- 相同 Key/相同 Payload：返回原结果。
- 相同 Key/不同 Payload：409。
- 执行中重复：返回处理中或等待原结果。

保存时间应覆盖 Client Retry Window。

## Error Model

稳定字段：Code、Human Message、Trace ID、Retryable、Field Error。内部 Exception/SQL 不暴露给 Client。

明确 Timeout/Unknown Outcome：客户端不能因为没收到 Response 就认为操作未执行。

## Deprecation

先测 Consumer Usage，再通知、提供 SDK/Adapter、设置 Sunset，最后移除。对外 API 的生命周期通常比内部实现长得多。

## 总结

API 演进的目标是让 Producer 和 Consumer 能独立发布。兼容添加、幂等写入、稳定错误和可观测弃用是核心机制。
