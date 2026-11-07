# 01. SQS Standard/FIFO、可见性超时与死信队列

## Standard

高吞吐、At-least-once、Best-effort Ordering。Consumer 必须 Idempotent。

## FIFO

按 Message Group 保序并提供 Deduplication Window 内的去重能力，吞吐与并行由 Group 数决定。FIFO 不消除业务 Side Effect 幂等需求。

## Visibility Timeout

Consumer 收到消息后暂时隐藏。处理时间超过 Timeout，消息可能被其他 Consumer 再取。设置略高于正常处理并支持 Heartbeat Extension；Crash 后等待过长会延迟恢复。

## Long Polling 与 Batch

Long Poll 减少空请求与成本；Batch Receive/Delete 提高吞吐。部分 Batch 失败要精确重试失败项。

## DLQ

超过 `maxReceiveCount` 进入 DLQ。设置 Alarm、Owner、Triage、Retention 和 Redrive 工具。DLQ 不是永久存档；修复根因后再重放。

## 安全与容量

Queue Policy、KMS、VPC Endpoint、消息大小和 Retention。大型 Payload 放 S3，Queue 传 Pointer，并处理两者生命周期。

## 总结

SQS 解耦 Producer/Consumer 并吸收峰值。Visibility、幂等、DLQ 与 Message Age 监控决定可靠性。
