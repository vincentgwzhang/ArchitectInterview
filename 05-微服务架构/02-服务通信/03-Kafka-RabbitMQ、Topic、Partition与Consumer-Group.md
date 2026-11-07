# 03. Kafka/RabbitMQ、Topic、Partition 与 Consumer Group

## Kafka

Distributed Log，Event 按 Partition 追加并可重放。适合高吞吐 Event Streaming、CDC 和多个独立 Consumer。

Partition 是并行与顺序单位。同一 Aggregate 用相同 Key 可保持 Partition 内顺序；全局顺序代价很高。Consumer Group 内一个 Partition 同时由一个 Consumer 实例处理，因此 Partition 数限制并行度。

## RabbitMQ

传统 Message Broker，Exchange/Queue/Routing 灵活，适合 Work Queue、复杂路由和低延迟消息。消息通常消费后 Ack 删除，重放模型不如 Kafka Natural。

## 选择

- 需要长期 Retention、Replay、Stream：Kafka。
- 任务分发、灵活 Routing、Per-message Ack：RabbitMQ。
- 云上简单 Queue：也可考虑 SQS 等 Managed Service。

## Kafka 设计

- Topic 按业务语义与 Retention。
- Key 避免 Hot Partition。
- Replication/Min ISR 保护 Durability。
- Consumer Commit 与业务处理协调。
- Schema Registry 管理 Compatibility。

## Rabbit 设计

Ack、Prefetch、Dead Letter、TTL、Quorum Queue 和 Poison Message。Consumer 必须幂等，Connection/Channel 有界。

## 运营

监控 Lag、Oldest Message Age、Partition Skew、Under-replicated、Rebalance、Redelivery 和 DLQ。Lag 低不一定业务正确，还需处理成功 Metric。

## 总结

Broker 选择来自消息生命周期、Replay、Routing 和运营模型。Partition/Queue 设计直接决定 Ordering、Parallelism 与故障行为。
