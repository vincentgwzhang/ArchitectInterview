# 02. SNS、EventBridge 与事件路由

## SNS

Pub/Sub Topic 推送到 SQS、Lambda、HTTP、Email 等。适合简单 Fan-out、通知和高吞吐广播。Filter Policy 可减少无关消息。

## EventBridge

Event Bus 按内容 Pattern 路由 AWS/SaaS/自定义 Event，支持 Archive/Replay、Schema 和跨账户。适合企业事件集成与松耦合路由。

## 选择

- 简单广播、Mobile/Email、SQS Fan-out：SNS。
- 丰富内容路由、SaaS/AWS Event、跨账户 Bus：EventBridge。
- Work Queue：SQS。
- 高吞吐有序 Stream/Replay：Kinesis/MSK。

## Event Contract

包含稳定 Source、Type、ID、Time、Subject 和 Detail，版本化 Schema。不要把内部数据库 Row Change 直接当长期业务 Contract。

## Reliability

配置 Retry、DLQ、Destination、Archive 和 Consumer Idempotency。HTTP Target 需 Timeout/认证；跨账户使用 Resource Policy 与 Organization Condition。

## 运营

监控 Failed Invocation、Throttling、DLQ、Age 与 Rule 数。事件未命中 Rule 可能静默丢失，关键流程需端到端 Audit/Reconciliation。

## 总结

SNS 擅长 Fan-out，EventBridge 擅长事件路由。选择基于交付、过滤、Replay、吞吐和治理，而不是都叫 Event-driven。
