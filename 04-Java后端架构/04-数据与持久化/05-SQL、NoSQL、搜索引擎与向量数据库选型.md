# 05. SQL、NoSQL、搜索引擎与向量数据库选型

## 从访问模式选型

先定义数据关系、Transaction、Query、规模、Latency、Consistency、Retention 和运营能力，而不是先选产品。

## SQL

适合关系、约束、Join、ACID 与灵活 Query。PostgreSQL/MySQL 通常是业务系统默认起点，能覆盖比很多人想象更大的规模。

## Key-value / Document

适合已知 Key Access、大规模水平扩展或灵活 Document。需要围绕 Partition Key 建模，避免 Hot Partition 和跨分区 Query。Schema 灵活不等于没有 Schema。

## Wide-column / Time-series

适合高写入、时间或 Partition-oriented 数据，如 Metric/Event。Query Pattern 通常需要预先设计。

## 搜索引擎

适合 Full-text、相关性、Faceting 与复杂搜索。通常不是交易 Source of Truth，通过 CDC/Event 同步，并接受 Index Lag 与重建需求。

## Vector Database

适合语义近邻检索。需要 Metadata Filter、ANN Index、Embedding Version 和 Retrieval Eval。中小规模可先使用 PostgreSQL Vector Extension。

## Polyglot Persistence

每增加一种存储都会增加 Backup、Security、Skill、Monitoring 和一致性成本。只有明确需求超过现有数据库能力时才引入。

## 面试要点

说明 Source of Truth、同步方式、Failure、Rebuild、Consistency 和 TCO，而不只是列产品优点。

## 总结

最简单能满足 NFR 的存储通常最好。根据数据与访问模式选择，并为复制、演进和运营成本负责。
