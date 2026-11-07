# 03. RDS、Aurora、Read Replica 与 Multi-AZ

## RDS

托管 PostgreSQL/MySQL/SQL Server 等，AWS 管基础设施、Backup、Patch 部分工作；客户仍负责 Schema、Query、Index、Connection、权限和容量。

## Multi-AZ

提供高可用 Standby 与自动 Failover，主要用于 Availability，不是读取扩展。Failover 会中断连接，应用需 DNS Re-resolution、Retry 与幂等。

## Read Replica

异步复制，用于读取扩展、报表和部分 DR。存在 Replication Lag，不能用于要求 Read-after-write 的所有 Query。Replica Failure/Promotion 需设计。

## Aurora

计算与分布式存储分离，多副本 Storage、快速 Failover 和 Reader Endpoint。Aurora Serverless 适合变化负载，但需评估 Scaling、连接与 Cost。

## 运营

RDS Proxy 缓解 Connection Storm；Performance Insights/CloudWatch 查看 Query/Wait；Automated Backup + PITR；Snapshot 跨账户/区域。

## 迁移与升级

Blue/Green、Read Replica、DMS 可降低停机。Major Upgrade 前做兼容、性能与回滚计划。

## 总结

Multi-AZ 解决可用性，Read Replica 解决读取/部分 DR，Aurora 改变存储与 Failover 模型。应用仍要处理连接、Lag 和 Schema。
