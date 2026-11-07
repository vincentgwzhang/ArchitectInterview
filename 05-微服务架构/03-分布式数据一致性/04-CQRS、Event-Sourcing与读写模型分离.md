# 04. CQRS、Event Sourcing 与读写模型分离

## CQRS

Command Model 处理业务不变量，Query Model 针对读取优化。两者可以是同一数据库的不同 Model，也可以独立存储并通过 Event 更新。

适合读写需求差异大、复杂 Domain 或需要多个 Read View。简单 CRUD 使用 CQRS 可能只增加同步和维护成本。

## Event Sourcing

以不可变 Domain Event 作为 Source of Truth，当前状态通过 Replay/Fold 得到：

```text
AccountOpened -> MoneyDeposited -> MoneyWithdrawn
```

优势：完整历史、Temporal Query、可重建 Projection。代价：Event Schema 永久演进、Replay、Storage、Debug、Privacy 删除和团队学习。

## Projection

异步构建 Read Model，必须 Idempotent、可重建并监控 Lag。Query 可能暂时看不到刚提交的 Command，可提供 Read-your-writes、返回新状态或明确 Pending。

## Snapshot

长 Event Stream 可定期 Snapshot 加速加载，但 Snapshot 是 Cache，可从 Event 重建，并要兼容 Aggregate Version。

## Event 演进

不可修改已发生事实。使用 Upcaster、兼容 Reader 或新 Event Type。测试历史全部 Event Replay。

## 区分

CQRS 不要求 Event Sourcing；Event Sourcing 通常自然配合 CQRS，但两者不是同义词。

## 总结

CQRS 分离不同模型，Event Sourcing 保存历史事实。只有审计、复杂行为和重建价值超过运营成本时才采用。
