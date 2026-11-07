# 03. 数据库迁移、Expand-and-Contract 与向后兼容

## 为什么危险

Rolling Deployment 中新旧版本同时运行。直接 Rename/Drop Column 会让旧版本失败，并可能阻止 Rollback。

## Expand-and-Contract

1. Expand：添加新 Table/Column/Index，保持旧结构。
2. Deploy 兼容代码，可读旧写新或双写。
3. Backfill 历史数据，限速并校验。
4. 切换 Read Path。
5. 观察并停止旧写。
6. Contract：确认无 Consumer 后删除旧结构。

每一步独立发布并可恢复。

## 大表变更

使用 Online DDL、Concurrent Index、Batch Backfill、Progress Checkpoint。监控 Lock、Replication Lag、I/O 和 Transaction Log。避免单一巨大 Transaction。

## 双写

Application Dual Write 可能部分失败。若同一数据库可放一个 Transaction；跨系统使用 Outbox/CDC，并设置 Reconciliation 与退出计划。

## Migration Tool

Flyway/Liquibase Script 不可随意修改已在生产执行的 Migration；新建修正版本。Pipeline 中验证空库升级和历史版本升级。

## Rollback

Schema Down Migration 可能丢数据，通常更安全的是 Roll-forward。发布前定义 Code Rollback 在新 Schema 下仍可运行。

## 总结

数据库演进是兼容协议。通过 Expand、Backfill、Switch、Contract 小步完成，确保新旧版本并存和安全 Roll-forward。
