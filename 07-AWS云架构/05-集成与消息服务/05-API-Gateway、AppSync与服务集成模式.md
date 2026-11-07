# 05. API Gateway、AppSync 与服务集成模式

## API Gateway

提供 REST/HTTP/WebSocket Endpoint、Authentication、Throttle、Stage 和 Lambda/AWS Service Integration。HTTP API 成本低、功能简；REST API 提供更丰富 Gateway Feature。

## AppSync

托管 GraphQL，支持 Resolver、Subscription、Caching 和多 Data Source。适合多客户端灵活查询与实时更新。需要治理 Query Depth/Complexity、N+1、Field-level Authorization 和 Schema 演进。

## Integration

- Lambda Proxy：灵活，增加 Function/Cold Start。
- Direct AWS Service：少代码，但 Mapping/错误处理复杂。
- HTTP/VPC Link：连接 ALB/NLB/Service。
- AppSync Resolver/Pipeline：组合 Data Source。

## BFF

Gateway/AppSync 可作为入口，但复杂 Domain Orchestration 应放在明确 BFF/Application Service，避免 Mapping Template 成为隐藏业务层。

## 安全

Cognito/OIDC/IAM/Auth Lambda，WAF、Rate Limit 和 Resource Policy。Gateway Authentication 不替代 Backend Resource Authorization。

## 运营

Access Log、Trace、Latency 分解、Usage/Quota、Custom Domain 和 Canary Stage。注意 Payload/Timeout Limit 与成本。

## 总结

API Gateway 适合 HTTP 边缘治理，AppSync 提供 GraphQL/实时能力。集成方式应平衡灵活性、延迟、可测试与业务边界。
