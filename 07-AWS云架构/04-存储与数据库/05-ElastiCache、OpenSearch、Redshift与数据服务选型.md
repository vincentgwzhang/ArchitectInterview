# 05. ElastiCache、OpenSearch、Redshift 与数据服务选型

## ElastiCache

Redis/Valkey 或 Memcached 托管 Cache。Redis/Valkey 支持数据结构、Replication、Persistence；Memcached 简单分布式 Cache。Cache 不是默认 Source of Truth，设计 TTL、Invalidation、Hot Key 和 Failover。

## OpenSearch

全文搜索、日志和分析。通过 CDC/Event 从主数据库同步，接受 Index Lag。规划 Shard、Mapping、Refresh、冷热层与 Snapshot。Dynamic Mapping 失控会造成 Mapping Explosion。

## Redshift

列式 Data Warehouse，适合大规模 OLAP、BI。数据通过 ETL/ELT 导入，与交易数据库分离。设计 Distribution/Sort、Workload Management、Spectrum 和数据治理。

## 选择

- 亚毫秒 Key/Session/Cache：ElastiCache。
- Full-text/Faceting/Log Search：OpenSearch。
- 聚合分析/BI：Redshift。

不要用 OpenSearch 代替强一致交易库，也不要用 Redis 保存唯一不可恢复业务数据，除非明确启用并验证 Durability。

## 运营

Multi-AZ、Backup、Encryption、VPC、Fine-grained Access、容量与 Cost。Managed 不代表无需 Shard/Query/Data Lifecycle 管理。

## 总结

这些服务是专用 Read/Compute Model。主数据 Ownership、同步、Lag、重建和故障降级必须明确。
