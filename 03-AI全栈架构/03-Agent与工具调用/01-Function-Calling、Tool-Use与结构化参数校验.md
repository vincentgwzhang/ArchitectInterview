# 01. Function Calling、Tool Use 与结构化参数校验

## 基本流程

```text
User -> Model selects tool + arguments
     -> Application validates and authorizes
     -> Tool executes
     -> Result returns to Model
     -> Model produces answer or next tool call
```

Model 只提出调用建议，Application 才是 Security Boundary。

## Tool Schema

工具应职责单一、名称清楚、参数有严格 JSON Schema：

```json
{
  "name": "get_order",
  "parameters": {
    "type": "object",
    "required": ["orderId"],
    "properties": {
      "orderId": { "type": "string", "pattern": "^ord_[a-zA-Z0-9]+$" }
    },
    "additionalProperties": false
  }
}
```

描述应说明适用与不适用场景。不要提供一个能执行任意 SQL、Shell 或 HTTP 的万能 Tool。

## Validation

执行前检查：

- Schema 和长度。
- User Identity 与 Scope。
- Resource-level Authorization。
- Tenant Boundary。
- Business Rule。
- Idempotency。
- Rate Limit。
- 高风险操作 Approval。

模型生成合法参数不代表用户有权限。

## Tool Result

返回最小、结构化、带状态的结果，避免把内部 Stack Trace、Secret 或大量无关数据塞回 Context。

```json
{ "status": "found", "orderId": "ord_1", "state": "SHIPPED" }
```

外部 Tool Result 同样是不可信输入，可能包含 Prompt Injection，应作为 Data 隔离。

## Failure

定义 Validation Error、Not Found、Permission Denied、Retryable Failure 和 Unknown Outcome。对 Side Effect Tool 使用 Idempotency Key，Timeout 后查询最终状态，不能盲目重试。

## Observability

记录 Model/Prompt Version、Tool Name、参数摘要、Authorization Decision、Latency、Result、Approval 和 Trace ID；敏感参数需脱敏。

## 总结

Tool Calling 把概率推理连接到确定性系统。严格 Schema、Application Authorization、最小 Tool、幂等与审计是生产安全的核心。
