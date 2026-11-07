# 04. API、数据模型、核心流程与异常流程设计

## 学习目标

1. 从 Use Case 推导 API 和 Data Model。
2. 设计清晰、兼容、幂等且可运营的 API Contract。
3. 用核心流程验证组件协作和 Transaction Boundary。
4. 系统化设计 Timeout、Retry、Duplicate 和 Partial Failure。

## 一、从 Domain 开始

API 和数据库不是两个独立作业。它们都应服务于 Domain 和 Use Case。

以订单系统为例，先识别：

- Actor：Customer、Support Agent、Warehouse。
- Use Case：Create Order、Cancel Order、Ship Order。
- Entity：Order、OrderItem、Payment、Shipment。
- Invariant：已发货订单不能取消；Order Total 必须等于 Item Total 加税费。
- State Transition：Pending → Confirmed → Shipped → Delivered。

然后再决定 API Shape、Transaction Boundary 和 Storage Model。

## 二、API 设计

### 2.1 Resource 与行为

REST API 尽量围绕 Resource：

```http
POST   /orders
GET    /orders/{orderId}
GET    /orders?customerId=123&status=CONFIRMED
POST   /orders/{orderId}/cancellations
```

`POST /cancelOrder` 可以工作，但 `cancellations` 作为 Resource 更容易表达状态、权限和审计。

### 2.2 Request 与 Response

```json
{
  "customerId": "cus_123",
  "items": [
    { "productId": "prd_42", "quantity": 2 }
  ],
  "currency": "EUR"
}
```

Response 不应直接暴露内部 JPA Entity。Public Contract 与内部模型的变化频率不同，应通过 DTO 隔离。

### 2.3 错误模型

统一错误结构：

```json
{
  "code": "INSUFFICIENT_INVENTORY",
  "message": "One or more items are unavailable",
  "traceId": "7b9...",
  "details": [
    { "productId": "prd_42", "available": 1 }
  ]
}
```

错误码应稳定并面向消费者，不能要求客户端解析自然语言。

### 2.4 Idempotency

创建订单和支付必须考虑 Client Retry：

```http
POST /orders
Idempotency-Key: 8f40c...
```

Server 保存 `Idempotency-Key + Operation + Response`。相同请求重复到达时返回第一次结果，而不是创建第二个订单。

需要定义：

- Key Scope。
- Retention Time。
- 同 Key 不同 Payload 的处理。
- In-progress Request 的并发行为。

### 2.5 Versioning 与 Compatibility

优先使用向后兼容变化：

- 增加 Optional Field。
- 不改变已有 Field 语义。
- Enum Consumer 对未知值保持容错。
- 先让 Consumer 接受新旧格式，再改变 Producer。

Breaking Change 才需要 `/v2`、新 Media Type 或新 Endpoint。

## 三、数据模型设计

### 3.1 从 Access Pattern 开始

不要只从对象关系出发。需要列出主要 Query：

- 通过 `order_id` 读取订单。
- 查询用户最近 100 个订单。
- 查询待发货订单。
- 按时间范围进行财务对账。

这些 Query 决定 Primary Key、Index、Partition 和是否需要额外 Read Model。

### 3.2 关系模型示例

```text
orders
- id PK
- customer_id
- status
- currency
- total_amount
- version
- created_at
- updated_at

order_items
- id PK
- order_id FK
- product_id
- product_name_snapshot
- unit_price_snapshot
- quantity
```

订单应保存购买时的商品名称和价格 Snapshot，不能在展示历史订单时依赖当前 Catalog 数据。

### 3.3 数据不变量

尽可能在多层保护关键 Invariant：

- Domain Validation。
- Database Constraint。
- Unique Index。
- Optimistic Lock。
- Transaction。

例如防止重复外部订单：

```sql
UNIQUE (merchant_id, external_order_id)
```

Application Check 不能替代 Unique Constraint，因为并发请求之间存在 Race Condition。

### 3.4 Ownership

在 Microservice 中，Order Service 拥有 Order Data。其他 Service 不应直接写入它的表，而应通过 API 或 Event 交互。

可以发布必要的 Event，但不要把内部数据库 Schema 当作公共 Contract。

## 四、核心流程设计

创建订单的简化流程：

```text
1. Client 发送 Create Order + Idempotency Key
2. API 验证身份和输入
3. Application Service 检查 Idempotency Record
4. 获取商品价格并预留库存
5. 创建 Order Aggregate
6. Transaction 写入 Order + Outbox Event
7. 返回 Order ID 和状态
8. Outbox Publisher 发送 OrderCreated
9. 下游异步处理通知和履约
```

设计时应明确：

- 哪些步骤在 Critical Path？
- 哪些可以异步？
- Transaction 从哪里开始、在哪里结束？
- 哪个系统是 Source of Truth？
- 每一步失败时是什么状态？

## 五、异常流程设计

### 5.1 Validation Failure

输入无效应快速返回 4xx，不应进入下游处理。区分格式错误、业务规则错误和权限错误。

### 5.2 Timeout

Timeout 并不表示下游没有执行。支付请求超时后可能已经成功扣款。

处理策略：

- 使用 Idempotency Key。
- 查询操作状态。
- 进入 `PAYMENT_PENDING` 而不是立即判定失败。
- 通过 Webhook/Event 最终确认。

### 5.3 Retry

Retry 只适用于 Transient Failure，并应具备：

- Exponential Backoff。
- Jitter。
- Maximum Attempts。
- Overall Deadline。
- Idempotency。
- Retry Budget。

对 Validation Error 或明确的业务拒绝进行 Retry 没有意义。

### 5.4 Partial Failure

如果库存已预留但支付失败，需要释放库存。可以通过 Saga Compensation 完成。

必须考虑补偿也失败：

- 持久化 Saga State。
- 重试补偿。
- Dead Letter Queue。
- Alert 和人工处理入口。
- 定期 Reconciliation。

### 5.5 Duplicate 与 Out-of-order Event

消费者应假设 Event 可能重复或乱序：

- 使用 Event ID 去重。
- 业务操作使用自然 Idempotency Key。
- 保存 Entity Version。
- 忽略旧版本 Event。
- 同一 Aggregate 使用相同 Partition Key 保序。

### 5.6 Dependency Unavailable

根据业务重要性选择：

- Fail Fast。
- 返回 Cache/Stale Data。
- Queue for later。
- Disable optional feature。
- Manual fallback。

例如 Recommendation Service 故障不应阻断 Checkout，可以隐藏推荐区域；Payment 故障则必须停止订单确认。

## 六、状态机

复杂流程应显式建模状态和合法迁移：

```text
PENDING -> CONFIRMED -> SHIPPED -> DELIVERED
   |            |
   v            v
CANCELLED    CANCEL_REQUESTED
```

每次 Command 应检查：

- 当前状态是否允许操作。
- Caller 是否有权限。
- 是否已经处理过。
- 状态迁移和 Side Effect 是否原子或可靠关联。

显式状态机比散落的 `if` 更容易测试、监控和恢复。

## 七、Observability by Design

设计 API 与流程时同时定义：

- Correlation ID / Trace ID。
- Business ID，如 `orderId`、`paymentId`。
- Structured Log。
- Latency、Error、Retry、Queue Lag Metrics。
- Audit Event。
- 状态停留时间告警。

例如监控 `PAYMENT_PENDING > 10 minutes`，可以发现第三方回调丢失或处理失败。

## 八、Security by Design

- Authentication 与 Authorization 分开考虑。
- 验证 Resource Ownership，避免 IDOR。
- 不在 Log 中输出 Token、Card 或 PII。
- 对敏感 Field 加密或 Tokenization。
- 对高风险操作保留 Audit Trail。
- API Gateway 限流不能替代业务级权限。

## 九、面试检查清单

1. API 是否围绕核心 Use Case？
2. Error Model 是否稳定？
3. 创建和支付是否 Idempotent？
4. Data Model 是否支持主要 Access Pattern？
5. Source of Truth 和 Ownership 是否明确？
6. Transaction Boundary 在哪里？
7. Timeout 后如何判断最终状态？
8. Retry 是否安全并有上限？
9. Partial Failure 如何补偿？
10. 如何监控卡住的流程？

## 十、自测题

1. 为什么 Public API 不应该直接返回 JPA Entity？
2. Client Timeout 为什么不能直接理解为操作失败？
3. Application Check 为什么不能替代数据库 Unique Constraint？
4. 哪些 Side Effect 适合移出同步 Critical Path？
5. 如何处理重复且乱序的 Kafka Event？

## 十一、总结

- API、Data Model 和流程都应从 Use Case 与 Invariant 推导。
- API Contract 必须考虑错误、兼容性、Idempotency 和权限。
- Data Model 应围绕 Access Pattern、Ownership 与一致性设计。
- 核心流程需要明确 Critical Path、Transaction 和 Source of Truth。
- 异常流程不是附录；Timeout、Retry、Partial Failure 和恢复是分布式设计的主体。
