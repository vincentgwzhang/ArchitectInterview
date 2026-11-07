# 02. Saga 编排、事件协同与补偿事务

## Saga

把长业务 Transaction 拆成本地 Transaction 序列。每一步成功后触发下一步；失败时执行 Compensation。

```text
Reserve Inventory -> Authorize Payment -> Confirm Order
        ^ fail            ^ fail
Release Inventory      Release Inventory/Cancel
```

## Orchestration

中央 Orchestrator 发送 Command、保存状态、处理 Timeout 和 Compensation。流程清晰、易观测，但 Orchestrator 需高可用，不能承载所有 Domain Logic。

## Choreography

Service 订阅 Event 并自主响应，耦合松、无中央流程节点；参与者多时流程隐蔽、循环和排障困难。

简单 Fan-out 可 Choreography，长且关键流程通常 Orchestration 更可控。

## Compensation

补偿不是技术回滚：已发送邮件无法“未发送”，只能发更正；已扣款通过 Refund。定义语义、权限、时限和失败处理。

## 实现要求

- 持久 Saga State。
- Idempotent Step。
- Timeout/Retry。
- Correlation ID。
- Versioned Workflow。
- Manual Intervention。
- Reconciliation。

## Isolation 问题

Saga 中间状态对其他请求可见，可能发生脏业务决策。可用 Reservation、Semantic Lock、状态检查和补偿降低风险。

## 总结

Saga 用业务状态与补偿换取跨服务可用性。Orchestration 提供可见控制，Choreography 适合简单松耦合事件。
