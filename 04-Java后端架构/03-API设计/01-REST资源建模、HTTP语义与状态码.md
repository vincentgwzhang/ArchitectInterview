# 01. REST 资源建模、HTTP 语义与状态码

## Resource Modeling

以业务名词和关系设计 URI：

```http
POST /orders
GET /orders/{id}
POST /orders/{id}/cancellations
```

复杂业务动作可以建模为子资源或 Command Endpoint，不必为了“纯 REST”牺牲清晰性。

## HTTP Method

- GET：Safe、Idempotent，不产生业务 Side Effect。
- PUT：完整替换或在指定 URI 创建，Idempotent。
- PATCH：部分更新，语义需明确定义。
- POST：创建/Command，默认不保证 Idempotent。
- DELETE：目标状态上通常 Idempotent。

## 状态码

- 200/201/202/204：成功、创建、异步接受、无内容。
- 400：格式/通用请求无效。
- 401：未认证。
- 403：已认证但无权限。
- 404：不存在，某些安全场景也用于隐藏 Resource。
- 409：状态冲突。
- 422：语义/业务验证失败。
- 429：限流。
- 5xx：Server/Dependency 无法完成。

不要所有错误都返回 200，也不要把业务拒绝变成 500。

## Header 与缓存

使用 Location、ETag、If-Match、Cache-Control、Retry-After、Idempotency-Key 和 Correlation ID 表达标准语义。

## Error Contract

采用稳定 Problem Details：`type/code/title/status/detail/traceId/fieldErrors`。客户端依赖 Error Code，不解析 Message。

## 安全

URI/Query 可能进入 Log 和 Browser History，不放 Token 或敏感 PII。对 Resource 做 Ownership Authorization。

## 总结

REST 的价值是统一语义和可演进 Contract。正确 Method、Status、Header 与 Resource Boundary 能降低 Client/Server 耦合。
