# 04. DynamoDB 分区键、GSI、容量模式与 Global Tables

## 建模

DynamoDB 从 Access Pattern 开始。Partition Key 决定数据分布，Sort Key 支持同一分区内范围和层级。高 Cardinality、均匀流量的 Key 避免 Hot Partition。

## Index

GSI 提供不同 Partition/Sort Key，异步更新并消耗独立容量；LSI 与主表同 Partition，创建后不能新增。每个 Index 增加写放大和存储。

## 容量

- On-demand：弹性、简单，适合不可预测/早期。
- Provisioned + Auto Scaling：稳定负载可能更经济。

监控 Throttle、Consumed Capacity、Hot Key 和 Item Size。Adaptive Capacity 不能修复持续单一 Hot Key。

## 一致性

主表/LSI 可选 Strong Read（限制条件下）；GSI 和 Global Tables 为 Eventual。Conditional Write、Transaction 和 Version 防冲突。

## Global Tables

Multi-region Multi-active 复制，提供本地低延迟与 Region Resilience。并发写冲突按服务规则解决，业务应避免同一 Item 多 Region 写或设计可合并模型。

## 单表设计

可将多个 Entity 存一表提高 Query 效率，但抽象复杂、学习成本高。根据团队和 Access Pattern 决定，不把 Single-table 当教条。

## 总结

DynamoDB 成败取决于 Key 与访问模式。GSI、容量和 Global Tables 都应围绕明确 Query、热点与一致性设计。
