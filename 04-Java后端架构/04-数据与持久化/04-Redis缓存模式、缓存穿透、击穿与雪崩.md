# 04. Redis 缓存模式、缓存穿透、击穿与雪崩

## Cache 模式

- Cache Aside：应用读 Cache，Miss 读 DB 并回填；写 DB 后失效 Cache。
- Read/Write Through：Cache Layer 负责后端访问。
- Write Behind：异步落库，性能高但 Durability/一致性更复杂。

Cache Aside 最常见，但删除与并发会产生短暂旧值。必须定义 Staleness Budget。

## 穿透

大量不存在 Key 每次访问 DB。对策：Negative Cache、参数校验、Bloom Filter、Rate Limit。Negative TTL 要短，避免新建数据长期不可见。

## 击穿

热点 Key 过期瞬间大量请求打到 DB。对策：Request Coalescing/Single Flight、互斥重建、逻辑过期、后台刷新。

## 雪崩

大量 Key 同时过期或 Redis 故障。对策：TTL Jitter、分批预热、Multi-AZ、Circuit Breaker、DB Protection 和 Load Shedding。

## Hot Key 与 Big Key

Hot Key 造成单 Shard 压力；可本地 Cache、复制读取或拆分。Big Key 阻塞操作与网络，避免巨大 Collection，并使用渐进扫描。

## 正确性

Cache 不是 Source of Truth。Key 包含 Tenant/Version；序列化 Schema 可演进；权限相关 Response 不跨用户缓存。Redis Lock 需理解 Lease、Fencing Token 和故障语义。

## 监控

Hit Ratio、Miss、Eviction、Memory、Latency、Hot Key、Connection、Replication Lag 与 Backend QPS。

## 总结

Cache 以一致性和复杂性换性能。设计正常 Miss、热点失效和 Redis 整体故障三个路径，才能避免缓存成为事故放大器。
