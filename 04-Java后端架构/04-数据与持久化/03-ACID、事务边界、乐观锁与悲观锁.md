# 03. ACID、事务边界、乐观锁与悲观锁

## ACID

- Atomicity：全部成功或全部回滚。
- Consistency：Transaction 前后满足约束。
- Isolation：并发执行的可见行为受控。
- Durability：提交后数据在目标故障模型下保留。

ACID 不自动保证业务正确；Invariant 必须通过 Domain、Constraint 和正确 Transaction Boundary 表达。

## Transaction Boundary

通常一个 Application Use Case 一个本地 Transaction。把外部 HTTP、Kafka 和多个数据库放入同一普通 Transaction 不会获得原子性。

跨系统使用 Outbox、Saga、Idempotency 和 Reconciliation，而不是假设 `@Transactional` 能覆盖远程调用。

## Optimistic Lock

使用 Version：

```sql
UPDATE orders SET status=?, version=version+1
WHERE id=? AND version=?
```

受影响行数为 0 表示 Conflict。适合冲突较少、读多写少；Client 可重新读取、Merge 或返回 409。

## Pessimistic Lock

`SELECT ... FOR UPDATE` 在 Transaction 内锁定记录，适合短时、强协调且冲突较高的操作。代价是等待、Deadlock 和吞吐下降。

## 选择

- 用户编辑文档：Optimistic，允许冲突提示。
- 库存热点：可用条件原子更新、队列/分区所有权，不一定直接悲观锁。
- 小范围高价值序列操作：Pessimistic 可考虑。

## 总结

先定义 Invariant 和冲突概率，再选择 Isolation 与 Lock。保持 Transaction 小而明确，跨边界使用分布式一致性模式。
