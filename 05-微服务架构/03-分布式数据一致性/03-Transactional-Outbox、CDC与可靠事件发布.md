# 03. Transactional Outbox、CDC 与可靠事件发布

## Dual Write 问题

先写 DB 后发 Kafka，进程可能在两者之间崩溃；先发 Event 后写 DB 也可能产生不存在的业务事实。

## Outbox

业务记录与 Outbox Event 在同一本地 Transaction 写入：

```sql
BEGIN;
INSERT INTO orders ...;
INSERT INTO outbox(id, aggregate_id, type, payload) ...;
COMMIT;
```

Publisher 再将 Outbox 发送到 Broker。

## 发布方式

- Polling Publisher：简单、可控，但增加 DB Poll 和 Latency。
- CDC：读取 WAL/Binlog，吞吐高、侵入低，但需要 Connector 运维、Schema 和顺序治理。

## 语义

Outbox 通常提供 At-least-once，Publisher 在发送成功但标记前崩溃会重复。Consumer 必须幂等。

## Event 设计

Outbox 保存稳定 Integration Event，而不是让 CDC 直接把每次表行变化公开给所有 Consumer。后者泄露内部 Schema 并产生低语义事件。

## 运营

监控 Oldest Unpublished Age、Count、Publish Error、Connector Lag、Poison Record 和 Cleanup。Outbox 表按时间清理/Partition，但不能删除尚未确认事件。

## 总结

Outbox 通过本地 ACID 消除业务提交与事件记录之间的 Gap；CDC/Publisher 负责传输，幂等 Consumer 负责重复。
