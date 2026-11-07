# 02. Kafka/Kinesis、顺序性、重试与投递语义

## Stream

按 `recipientId` 或 Conversation ID 分区，保证单用户/会话内顺序并支持并行。热点广播不能全部使用同一 Key，可使用分层 Fan-out。

## 语义

采用 At-least-once + Idempotent Delivery。Message ID、Channel 和 Recipient 形成去重 Key；Client 也按 ID 去重。

## Retry

区分 Transient、Permanent 和 Rate Limit。Exponential Backoff + Jitter；按 Channel 建 Retry Topic/Queue，避免阻塞主 Partition。超过上限进入 DLQ。

## 顺序

Retry 可能打乱顺序。对强顺序会话使用 Sequence、单 Partition 或等待前序；普通通知可允许乱序并按 Timestamp 展示。

## 状态

保存 Accepted、Dispatched、ProviderAccepted、Delivered、Failed（若 Provider 支持）。不要把 Provider 接受误认为用户已读。

## 监控

Lag/Age、Redelivery、DLQ、Provider Error、Duplicate、End-to-end Delivery Latency。

## 总结

Broker 提供持久传输，端到端投递仍需分区、幂等、状态与重试设计。
