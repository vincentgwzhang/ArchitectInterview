# 05. REST、GraphQL、gRPC 与异步 API 的选择

## REST

成熟、可缓存、易调试，适合公共 API、资源操作和跨组织集成。复杂页面可能 Over/Under-fetch，但通过 BFF、Fields 和批量 API 可改善。

## GraphQL

Client 精确选择字段，适合多个 UI 聚合复杂 Graph。需要治理 Query Complexity、N+1/DataLoader、Authorization、Persisted Query、Cache 和 Schema Evolution。

GraphQL 不是数据库直通层；Resolver 必须遵守 Domain 与权限边界。

## gRPC

Protobuf、HTTP/2、强类型和 Streaming，适合内部低延迟 Service-to-service。Browser、公共可读性、Proxy 和版本治理需额外考虑。字段编号不可复用，删除字段应 `reserved`。

## 异步 API

Kafka/SQS/Event 适合解耦、削峰和长流程。代价是 Eventual Consistency、Duplicate、Ordering 和 Observability。使用 AsyncAPI/Schema Registry 管理 Contract。

## 选择维度

- Consumer 类型与网络边界。
- 同步结果是否必要。
- Latency/Throughput。
- Schema 与 Version。
- Streaming。
- Cache。
- Failure/Consistency。
- Team/Platform 能力。

同一系统可以组合：Public REST、Frontend GraphQL BFF、内部 gRPC、Domain Event Kafka。

## 面试要点

避免“GraphQL 比 REST 先进”。说明业务交互、失败语义、运营能力和组织 Contract。

## 总结

协议选择是 Communication Pattern 决策。同步/异步、Consumer 自主性和治理成本比技术流行度更重要。
