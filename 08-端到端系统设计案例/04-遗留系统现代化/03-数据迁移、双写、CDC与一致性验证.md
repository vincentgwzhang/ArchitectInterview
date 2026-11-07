# 03. 数据迁移、双写、CDC 与一致性验证

## Ownership

先决定新旧系统谁在每个阶段是 Source of Truth。没有明确 Ownership 的长期双写会产生不可解释冲突。

## 迁移步骤

1. Schema/Mapping 与 Data Quality Profile。
2. Bulk Backfill，Checkpoint/限速。
3. CDC 捕获增量。
4. Reconcile Count/Hash/Business Invariant。
5. Shadow/Dual Read。
6. 切写 Ownership。
7. 观察后停止旧写。

## Dual Write

应用同步双写会部分失败。优先 Outbox/CDC 或单一权威写入 + 异步投影。必须幂等、可重放并记录 Watermark。

## 验证

不仅比较 Row Count，还比较金额合计、状态分布、Foreign Key、时间范围和抽样业务结果。差异有自动修复/人工 Queue。

## Rollback

切换后若新系统已产生独有写入，简单切回会丢数据。提前设计 Reverse Sync、Freeze 或 Roll-forward。

## 总结

数据迁移是有状态协议。Backfill、CDC、Ownership、对账和回滚必须作为一个整体设计。
