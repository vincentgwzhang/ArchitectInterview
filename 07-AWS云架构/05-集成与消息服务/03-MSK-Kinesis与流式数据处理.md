# 03. MSK/Kinesis 与流式数据处理

## Amazon MSK

托管 Apache Kafka，兼容 Kafka Client/Ecosystem，适合已有 Kafka 技能、开放协议、复杂 Stream 和长 Retention。仍需规划 Broker、Partition、Storage、Upgrade 和 Consumer Lag。

## Kinesis Data Streams

AWS Native Stream，按 Shard/On-demand 容量，集成 Lambda、Firehose 和 IAM。适合 AWS 生态实时数据，运营相对简化，但 API/Ecosystem 与 Kafka 不同。

## 关键设计

- Partition Key 分布与 Ordering。
- Retention/Replay。
- Producer Ack/Retry。
- Consumer Checkpoint。
- Enhanced Fan-out/Consumer Group。
- Schema Registry。
- At-least-once 与 Idempotency。

## Stream Processing

窗口、Watermark、Late Event、State Store、Checkpoint 和 Exactly-once Boundary。业务时间（Event Time）与处理时间不同。

## 选择

Kafka Compatibility、多平台生态、自定义控制：MSK。AWS Native、快速集成、少运维：Kinesis。用真实吞吐、Record Size、Consumer 数和 TCO 比较。

## 运营

Lag/Age、Hot Partition、Throttle、Broker/Shard Capacity、Checkpoint、DLQ 和 Data Quality。Stream 可重放，但 Side Effect 仍需幂等。

## 总结

MSK 与 Kinesis 都提供持久 Stream，差异在生态、运营与容量模型。Partition 设计和消费语义比服务名称更关键。
