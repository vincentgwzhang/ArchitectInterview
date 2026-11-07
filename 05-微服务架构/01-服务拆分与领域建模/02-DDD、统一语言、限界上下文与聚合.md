# 02. DDD、统一语言、限界上下文与聚合

## Ubiquitous Language

业务与技术团队共同使用准确术语。相同词在不同 Context 可能含义不同，例如 Sales 的 Customer 与 Support 的 Customer Model 不必强行统一。

## Bounded Context

限定某个 Domain Model 适用范围，内部保持语言和规则一致。Context 之间通过明确 Contract 交互，是 Service Boundary 的重要候选，但不要求一对一。

## Context Mapping

关系包括 Partnership、Customer/Supplier、Conformist、Anti-corruption Layer、Published Language。ACL 可防止外部 Legacy Model 污染内部 Domain。

## Entity 与 Value Object

- Entity：由 Identity 与生命周期定义，如 Order。
- Value Object：由值定义、通常 Immutable，如 Money、Address。

Value Object 可集中保护单位、格式和业务规则。

## Aggregate

Aggregate 是一致性边界，Root 控制内部不变量。一次 Transaction 通常只修改一个 Aggregate；跨 Aggregate 使用 Event/Process Manager。

不要把整个 Domain 变成巨大 Aggregate，否则锁、加载和并发冲突严重。只把必须立即一致的对象放在一起。

## Domain Event

表示已发生业务事实，如 `OrderConfirmed`。它不是 CRUD 表变化，也不应泄露全部内部 Entity。

## 面试要点

DDD 不是目录命名或大量 Pattern，而是用 Domain 知识和边界控制复杂度。说明如何通过 Event Storming、业务变化和 Invariant 发现边界。

## 总结

统一语言帮助建模，Bounded Context 隔离模型，Aggregate 保护局部一致性，Context Map 定义协作关系。
