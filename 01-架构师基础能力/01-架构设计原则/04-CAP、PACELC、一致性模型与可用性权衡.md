# 04. CAP、PACELC、一致性模型与可用性权衡

## 学习目标

1. 准确解释 CAP，而不是简单背诵“三选二”。
2. 理解 PACELC 为什么补充了正常状态下的延迟权衡。
3. 区分 Strong Consistency、Eventual Consistency 等一致性模型。
4. 能够根据业务不变量选择合适的一致性策略。
5. 在系统设计面试中清楚表达 Availability 与 Consistency 的权衡。

## 一、CAP 的基本概念

CAP 讨论的是分布式系统发生 Network Partition 时的行为。

- Consistency（C）：所有节点在同一时刻看到相同的数据，通常接近 Linearizability。
- Availability（A）：每个发送到未故障节点的请求都能在有限时间内得到非错误响应。
- Partition Tolerance（P）：系统在节点之间消息丢失或延迟的情况下仍能继续运行。

在真实分布式系统中，Network Partition 无法完全避免。因此关键选择是：

> 当 Partition 发生时，系统选择拒绝部分请求来保护 Consistency，还是继续响应并接受数据可能不一致？

这就是 CP 与 AP 的核心区别。

## 二、为什么“三选二”不够准确

“CAP 三选二”容易产生误解：

1. Partition 不是可以随意关闭的 Feature。
2. 选择不是整个系统永久属于 CP 或 AP。
3. 同一个系统的不同操作可以做不同选择。
4. Network Partition 之外，系统仍然需要权衡 Latency 和 Consistency。
5. Availability 在 CAP 中有严格含义，不等同于常见的 SLA 百分比。

例如订单系统可以对“扣减库存”选择强一致和失败响应，对“商品评价数量”选择最终一致和持续可用。

架构决策的单位应是具体 Operation 或 Business Invariant，而不是简单给整个系统贴标签。

## 三、CP 与 AP 行为

假设 Region A 和 Region B 之间发生 Network Partition。

### 3.1 CP 选择

系统拒绝无法确认一致性的写入，或者只允许 Leader 所在一侧继续处理。

适合场景：

- 银行余额。
- 唯一用户名注册。
- 库存最后一件商品。
- 权限和安全策略更新。

代价是部分请求失败或等待，降低 Availability。

### 3.2 AP 选择

两个 Region 都继续接受请求，网络恢复后进行合并或冲突解决。

适合场景：

- 点赞计数。
- 浏览历史。
- 部分购物车操作。
- Analytics Event。
- 非关键用户偏好。

代价是客户端可能暂时看到旧值、不同值，甚至需要解决冲突。

## 四、PACELC

PACELC 补充了 CAP：

```text
If Partition: Availability vs Consistency
Else: Latency vs Consistency
```

即使网络正常，为了 Strong Consistency，系统也可能需要跨节点或跨 Region 同步确认，从而增加 Latency。

例如：

- 跨 Region Quorum Write 提高一致性，但增加写入延迟。
- 读取本地 Replica 延迟低，但可能读到旧数据。
- 读取 Leader 更一致，但跨 Region 请求更慢。

PACELC 提醒架构师：一致性成本并不只在故障时出现，它也存在于系统的日常路径中。

## 五、一致性模型

### 5.1 Strong Consistency

写入成功后，任何后续读取都返回最新值。工程讨论中这个术语有时比较宽泛，需要继续澄清具体保证。

适合强业务不变量，但通常需要同步协调。

### 5.2 Linearizability

每个操作看起来像在调用和返回之间的某个瞬间原子完成，并且尊重真实时间顺序。

它适合锁、Leader Election、唯一性约束等需要实时顺序的场景。

### 5.3 Sequential Consistency

所有操作表现为某个统一顺序，并保持每个客户端自己的操作顺序，但这个统一顺序不一定符合真实时间顺序。

### 5.4 Causal Consistency

有因果关系的操作必须按相同顺序被观察；没有因果关系的操作可以采用不同顺序。

例如，用户先发帖子再回复帖子，其他用户不应该先看到回复却看不到原帖。

### 5.5 Eventual Consistency

在没有新写入的情况下，所有 Replica 最终会收敛到相同状态。它没有保证收敛需要多长时间，也不自动保证冲突能正确解决。

### 5.6 Session Guarantees

对用户体验很有价值的保证包括：

- Read Your Writes：用户能读到自己的最新写入。
- Monotonic Reads：用户不会先看到新版本，随后又看到旧版本。
- Monotonic Writes：同一用户的写入按照发出顺序应用。
- Writes Follow Reads：基于已读版本的写入不会应用到更旧的状态。

一个系统即使整体是 Eventual Consistency，也可以为单个 Session 提供这些保证。

## 六、复制与 Quorum

假设：

- `N`：Replica 数量。
- `W`：写入成功所需确认的 Replica 数量。
- `R`：读取所需响应的 Replica 数量。

当 `R + W > N` 时，读写集合至少有一个重叠节点，有助于读取最新数据。

例如：

```text
N = 3, W = 2, R = 2
```

但 Quorum 公式不自动等于 Linearizability。仍需考虑：

- 并发写入。
- Clock 和版本比较。
- Sloppy Quorum。
- Read Repair。
- Last-write-wins 的数据丢失风险。
- 节点故障和恢复过程。

## 七、业务层的一致性策略

### 7.1 先识别 Business Invariant

不要从“数据库支持什么”开始，而应先问：

- 哪些状态绝不能同时成立？
- 哪些数据允许短暂过期？
- 不一致最多能持续多久？
- 错误结果能否补偿？
- 用户是否必须立即看到自己的修改？

例如“库存不能为负”是一个 Invariant；“商品页显示的销量允许延迟 30 秒”是一个 Staleness Budget。

### 7.2 Transaction

单个数据库内可以使用 ACID Transaction 保证相关修改原子完成。

跨 Service 后，通常要在以下方案中选择：

- Distributed Transaction / 2PC。
- Saga。
- Transactional Outbox。
- Eventual Consistency 与补偿。

每种方案都涉及 Availability、Latency 和复杂度权衡。

### 7.3 Saga

创建订单可能包含：

```text
Reserve Inventory -> Authorize Payment -> Confirm Order
```

如果 Payment 失败，可以执行 `Release Inventory`。这不是数据库意义上的 Rollback，而是业务补偿。

Saga 的挑战包括：

- 补偿本身可能失败。
- 操作必须 Idempotent。
- 用户可能看到中间状态。
- 流程状态需要持久化和监控。

### 7.4 Transactional Outbox

业务数据与待发送 Event 写入同一个本地 Transaction：

```text
BEGIN
  INSERT INTO orders ...
  INSERT INTO outbox_events ...
COMMIT
```

后台 Publisher 再把 Outbox Event 发送到 Kafka。它解决数据库提交成功但 Event 发布失败的不一致窗口。

消费者仍需处理重复消息，因此要设计 Idempotency。

## 八、冲突解决

AP 系统需要明确冲突如何处理：

- Last Write Wins：简单，但可能覆盖有效更新。
- Version Vector：可以识别并发更新，但实现复杂。
- Application Merge：由业务规则合并。
- CRDT：通过特定数据结构保证并发更新可合并。
- User Resolution：让用户选择正确版本。

购物车商品集合可以做 Union，但账户余额不能简单合并。冲突策略必须理解业务语义。

## 九、常见误区

### 9.1 Eventual Consistency 等于“不保证一致”

错误。它保证在没有新更新时最终收敛，但需要明确收敛机制、冲突策略和时间目标。

### 9.2 Kafka 可以自动解决分布式事务

错误。Kafka 可以可靠传递 Event，但数据库写入与 Event 发布、消费者副作用、重复消息仍需处理。

### 9.3 强一致永远更好

错误。强一致可能增加 Latency、降低 Availability 和提高成本。关键是业务是否需要。

### 9.4 NoSQL 一定是 AP，SQL 一定是 CP

错误。具体行为取决于产品、配置、拓扑和每个 Operation 的 Consistency Level。

### 9.5 Retry 可以解决所有失败

错误。对非 Idempotent 操作盲目 Retry 可能造成重复扣款或重复订单。

## 十、系统设计示例：全球购物车

需求：用户从不同设备、不同 Region 修改购物车，希望低延迟和高可用。

可能设计：

1. 每个 Region 接受本地读写。
2. 修改通过异步复制传播。
3. 商品集合使用可合并结构。
4. 每个 Item 保存版本或更新时间。
5. 对同一商品数量冲突定义明确业务规则。
6. Checkout 前重新验证价格与库存。

购物车本身可以偏 AP，但 Checkout 和库存预留需要更强一致性。这展示了同一业务流程中按 Operation 选择不同模型。

## 十一、面试回答框架

回答一致性问题时可以采用：

1. 定义业务 Invariant。
2. 区分正常状态与 Partition 状态。
3. 说明允许的 Staleness 和失败行为。
4. 选择同步或异步复制方式。
5. 说明冲突、Retry、Idempotency 和补偿。
6. 给出 Metrics、Alert 和 Reconciliation Job。

示例：

> 商品描述允许秒级 Eventual Consistency，因此优先本地读取和 Availability；库存扣减不能导致超卖，因此写入由单一权威节点或具有条件更新能力的存储处理。发生 Partition 时，无法确认库存的请求失败，而不是接受风险写入。

## 十二、自测题

1. 为什么 CAP 不能简单理解为任意选择三个属性中的两个？
2. PACELC 增加了什么重要视角？
3. Eventual Consistency 与 Causal Consistency 有什么区别？
4. `R + W > N` 为什么仍不自动保证 Linearizability？
5. 为什么同一个系统可以同时具有 CP 和 AP 风格的操作？

## 十三、总结

- CAP 关注 Network Partition 发生时 Consistency 与 Availability 的选择。
- PACELC 进一步指出正常情况下 Latency 与 Consistency 也需要权衡。
- 一致性不是二元选项，而是一组具有不同保证的模型。
- 决策应以具体 Operation、Business Invariant 和 Staleness Budget 为单位。
- Eventual Consistency 需要 Idempotency、冲突解决、补偿和可观测性支持。
- 架构师必须解释业务后果，而不只是说系统属于 CP 或 AP。
