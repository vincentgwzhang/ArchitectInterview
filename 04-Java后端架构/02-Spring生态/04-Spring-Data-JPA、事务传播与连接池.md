# 04. Spring Data JPA、事务传播与连接池

## JPA

JPA 管理 Entity、Persistence Context、Dirty Checking 和关系映射。Entity 是持久化模型，不应直接作为 Public API DTO。

常见问题：N+1 Query、意外 Lazy Loading、巨大 Object Graph、错误 Cascade 和 `equals/hashCode`。使用 Fetch Join、Entity Graph、Projection 与 SQL Metric 验证，不要把所有关系设为 EAGER。

## Transaction

`@Transactional` 通常通过 Proxy 生效，自调用与非 Public Method 可能绕过。Transaction Boundary 应位于 Application Use Case，时间尽量短，禁止持有数据库 Transaction 等待用户或慢外部 API。

传播语义：

- REQUIRED：加入现有或新建。
- REQUIRES_NEW：挂起现有并新建，可能额外占连接。
- MANDATORY、SUPPORTS 等用于更明确 Contract。

默认 Runtime Exception 回滚；Checked Exception 需按规则配置。理解行为而不是只背 Annotation。

## Isolation 与 Lock

根据 Lost Update、Non-repeatable Read、Phantom 等风险选择 Isolation。Optimistic Lock 使用 Version 防止覆盖；Pessimistic Lock 适合短时高价值冲突，但增加等待和 Deadlock。

## Connection Pool

Pool Size 不是越大越好。总连接 = Instance × Pool，必须低于数据库安全容量。监控 Active、Idle、Wait、Timeout 和 Transaction Duration。

## Open Session in View

可能让 Web Layer 意外触发 Query，隐藏 N+1 和 Transaction Boundary。复杂系统通常显式关闭并在 Service 层完整加载 DTO 所需数据。

## 总结

JPA 提高开发效率，但必须控制 Query、Entity Boundary、Transaction 和连接。性能与一致性问题通常来自隐式行为。
