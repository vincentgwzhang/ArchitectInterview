# 05. 架构决策记录（ADR）与技术债管理

## 学习目标

1. 理解 Architecture Decision Record（ADR）的目的和边界。
2. 能够编写一份简洁、可追溯的 ADR。
3. 区分技术债、缺陷、风险和普通待办事项。
4. 建立技术债的识别、量化、排序和偿还机制。
5. 在架构治理中平衡交付速度与长期健康度。

## 一、为什么架构决策需要记录

系统会留下代码和配置，却不会自动留下当时的上下文。几个月后，团队可能只知道“我们使用 Kafka”，却不知道：

- 当时要解决什么问题？
- 考虑过哪些替代方案？
- 有哪些时间、预算或合规约束？
- 为什么接受某些缺点？
- 什么条件变化后需要重新评估？

缺少这些信息，后来的工程师可能把一个有意的权衡误认为错误，也可能重复已经讨论过的争论。

ADR 用小型文档记录重要架构决策及其背景。它的价值是保存 Decision Context，而不是生成大量形式化文档。

## 二、什么是 ADR

一份 ADR 通常记录：

- Context：为什么需要决策。
- Decision：最终选择什么。
- Alternatives：考虑过哪些方案。
- Consequences：正面和负面后果。
- Status：Proposed、Accepted、Deprecated、Superseded 等。

ADR 应该和代码一起进入 Version Control，因为架构决策与实现需要共同演进。

建议目录：

```text
docs/
└── adr/
    ├── 0001-use-postgresql-for-orders.md
    ├── 0002-adopt-transactional-outbox.md
    └── 0003-use-opentelemetry.md
```

编号一旦使用就不应该重新排序。被替代的 ADR 也应保留，用于维护历史链路。

## 三、哪些决策应该写 ADR

适合记录的决策通常具有以下一个或多个特征：

- 跨多个团队或 Module。
- 未来替换成本高。
- 影响重要 NFR。
- 涉及 Security、Compliance 或 Data Governance。
- 存在多个合理选项和明显权衡。
- 引入新的 Platform、Database、Framework 或 Protocol。
- 改变 Service Boundary、数据所有权或部署拓扑。

示例：

- 选择 Modular Monolith 还是 Microservices。
- 选择 REST、gRPC 还是 asynchronous event。
- 是否采用 Multi-region Active-Active。
- 使用自建 Kafka 还是 Amazon MSK。
- AI 应用使用托管 Model API 还是自托管 Model。

不需要为每个 Class 名称或小型 Refactoring 写 ADR。记录成本应与决策影响匹配。

## 四、ADR 模板

```markdown
# ADR-0007：使用 Transactional Outbox 发布订单事件

- Status: Accepted
- Date: 2026-11-03
- Owners: Order Platform Team
- Deciders: Architecture Group, Order Team

## Context

订单写入 PostgreSQL 后需要向 Kafka 发布 OrderCreated Event。
直接先写数据库再发送 Event 存在部分失败窗口，可能导致订单已创建但下游没有收到消息。

## Decision Drivers

- 不允许丢失已提交订单的 Event。
- 允许至少一次投递。
- 不引入 2PC。
- 团队已有 PostgreSQL 和 Kafka 运维经验。

## Considered Options

1. Database + Kafka Dual Write。
2. Distributed Transaction / 2PC。
3. Transactional Outbox + CDC。

## Decision

业务数据和 Outbox Record 在同一 Database Transaction 中写入，
由 CDC Connector 把 Outbox Record 发布到 Kafka。

## Consequences

### Positive

- 已提交业务记录对应的 Event 不会因进程崩溃而丢失。
- 不需要跨 PostgreSQL 和 Kafka 使用 2PC。

### Negative

- 消费者必须支持 Idempotency。
- 需要监控 Outbox Lag 和 CDC Connector。
- Event 只能保证至少一次投递。

## Revisit When

- Outbox Lag 无法满足 2 秒以内的 SLO。
- CDC 运维成本超过可接受范围。
```

## 五、如何写出高质量 ADR

### 5.1 从问题开始，而不是从技术开始

较差的 Context：

> 我们需要使用 Kafka。

更好的 Context：

> 订单创建请求不应该等待通知和 Analytics 处理；峰值为 5,000 RPS，下游可能短暂不可用，但 Event 不能丢失。

后者描述问题和约束，没有提前锁定答案。

### 5.2 明确 Decision Drivers

Driver 可能包括：

- Latency。
- Throughput。
- Consistency。
- Availability。
- Security 和 Compliance。
- Team Skill。
- Delivery Deadline。
- Total Cost of Ownership。
- Vendor Lock-in。

如果没有 Driver，替代方案无法进行有意义的比较。

### 5.3 记录被否决方案

ADR 不只是记录最终答案。替代方案及其否决原因往往是最有价值的信息。

不要把未选方案写成明显错误的 Straw Man。应该公平描述每个方案的优势，以及它为什么不适合当前 Context。

### 5.4 同时写正面与负面后果

没有代价的技术决策通常不真实。采用 Microservices 可能获得独立部署，也会引入网络故障、数据一致性和 Platform 成本。

### 5.5 设置重新评估条件

决策是在特定 Context 下作出的。可以写清以下触发条件：

- 流量增长到某个水平。
- 团队数量发生变化。
- Vendor Price 或 License 改变。
- 新的 Compliance Requirement 出现。
- 当前方案违反 SLO。

这使决策既稳定又不僵化。

## 六、ADR 生命周期

推荐状态：

```text
Proposed -> Accepted -> Deprecated
                    -> Superseded by ADR-xxxx
         -> Rejected
```

重要原则：

- Accepted ADR 不应被悄悄修改成另一项决策。
- 如果决策改变，创建新 ADR 并引用旧 ADR。
- 可以修正拼写或补充链接，但要保留历史真实性。
- ADR Review 应与相关代码变更在同一流程中完成。

## 七、ADR 与其他文档的关系

- ADR：为什么选择某个方案。
- Architecture Diagram：系统结构是什么。
- API Specification：接口契约是什么。
- Runbook：发生问题时如何操作。
- RFC：在决策前进行更完整的讨论。
- Postmortem：事故发生了什么以及如何改进。

这些文档目的不同，不能互相完全替代。

## 八、什么是技术债

Technical Debt 是为了更快获得短期收益而选择了一个长期成本更高的实现，从而产生未来需要支付的额外成本。

这个比喻包含：

- Principal：彻底修复所需成本。
- Interest：在债务存在期间，每次开发、测试、部署和排障增加的成本。

技术债不一定是错误。有意识地承担可控技术债，可以帮助产品更快验证市场。真正危险的是团队不知道债务存在，或者没有偿还策略。

## 九、技术债的类型

### 9.1 Deliberate Debt

团队知道理想方案，但为了 Deadline 或验证假设暂时选择较简单实现。

必须记录：原因、风险、Owner、偿还条件和最晚日期。

### 9.2 Accidental Debt

由于经验不足、需求变化或早期假设失效产生。

例如早期数据量很小，后来 Query Pattern 变化，原有 Schema 不再合适。

### 9.3 Architecture Debt

- Service Boundary 不合理。
- 共享数据库。
- 循环依赖。
- 缺少故障隔离。
- 无法独立部署。

### 9.4 Code Debt

- 重复业务逻辑。
- 复杂 method。
- 过时 Framework。
- 缺少测试。
- 不清晰命名。

### 9.5 Operational Debt

- 手工部署。
- 缺少 Metrics 和 Alert。
- 没有 Runbook。
- 无法快速 Rollback。
- 备份没有恢复演练。

### 9.6 Security Debt

- 过期依赖。
- 长期有效的 Access Key。
- 权限过宽。
- 数据未加密。
- Critical Vulnerability 未处理。

Security Debt 通常具有更严格的处理优先级，不能只作为普通 Backlog。

## 十、如何记录技术债

一个可操作的 Technical Debt Item 应包含：

```text
Title: Order Service 直接访问 Inventory Table
Type: Architecture Debt
Owner: Commerce Platform Team
Introduced: 2026-Q2
Reason: 为赶上迁移 Deadline 的临时方案
Impact: Inventory Schema 每次变更都需要协调发布
Interest: 平均每月增加约 3 engineer-days
Risk: 独立部署失败、数据所有权不清晰
Remediation: 通过 Inventory API/Event 替代跨库访问
Estimate: 15 engineer-days
Target: 2027-Q1
Trigger: Inventory Team 开始独立发布前必须完成
```

只有“重构 Order Service”这样的描述很难排序，也无法验证是否完成。

## 十一、技术债排序

可以从以下维度评分：

- Business Impact：是否阻碍新 Feature 或收入。
- Operational Risk：是否容易导致事故。
- Security/Compliance Risk：是否违反强制要求。
- Interest：持续拖慢团队的程度。
- Remediation Cost：偿还成本。
- Time Criticality：风险是否快速增长。

一个简单的优先级思路是：

```text
Priority ≈ (Impact × Probability + Ongoing Interest) / Remediation Cost
```

公式不是为了制造虚假精度，而是迫使团队显式比较价值、风险和成本。

## 十二、偿还技术债的策略

### 12.1 Boy Scout Rule

每次修改代码时，顺手改善附近的小问题。适合低风险、局部债务。

### 12.2 Capacity Allocation

每个 Iteration 保留一定比例用于 Reliability、Upgrade 和 Refactoring。比例应根据债务和业务阶段调整，而不是固定迷信某个数字。

### 12.3 Feature-funded Refactoring

在实现新 Feature 时，把其依赖的技术债纳入 Feature Scope。这通常比单独申请“大重构项目”更容易体现业务价值。

### 12.4 Dedicated Initiative

当债务已跨越多个团队或造成重大风险时，需要明确的 Program、Owner、Milestone 和 Executive Support。

### 12.5 Strangler Fig

对于大型遗留系统，逐步把能力迁移到新边界，而不是一次性 Rewrite。

## 十三、避免技术债清单失效

常见失败模式：

- 只登记，不评审。
- 没有 Owner。
- 没有业务影响描述。
- 把所有不喜欢的代码都叫技术债。
- 一直增加新债，没有删除已解决或已接受的项目。
- 只关注 Code Debt，忽略 Operational 和 Security Debt。

建议建立周期性治理：

1. 每月由 Team Review 自己的债务。
2. 每季度评审跨团队 Architecture Debt。
3. Critical Security Debt 进入独立 SLA。
4. 使用 Dashboard 观察新增、偿还和逾期趋势。
5. 在 Planning 中把债务与业务目标一起讨论。

## 十四、ADR 与技术债的结合

ADR 可以清楚记录团队为什么主动承担技术债。

例如：

```text
Decision:
第一版使用单 Region 部署，以在 8 周内发布 MVP。

Accepted Risk:
Region Failure 时服务不可用。

Mitigation:
每日跨 Region Backup，恢复流程每月演练。

Revisit Trigger:
付费客户达到 100 家，或合同要求 99.95% SLA。
```

这种记录让“暂时方案”具有边界、Owner 和退出条件，而不是无限期存在。

## 十五、面试场景

问题：业务要求两个月上线，但理想架构需要四个月，你会怎么做？

推荐回答：

1. 明确两个月内必须交付的业务能力和 NFR。
2. 识别不能妥协的 Security、Compliance 和 Data Integrity 要求。
3. 设计满足当前规模的最小可行架构，同时保留演进边界。
4. 通过 ADR 记录选择、替代方案、风险和触发条件。
5. 将产生的技术债量化，指定 Owner 和偿还计划。
6. 加入 Monitoring、Backup、Feature Flag 等风险缓解措施。
7. 与 Product 和 Engineering 共同接受决策，而不是由架构师单方面决定。

## 十六、自测题

1. 哪些决策值得写 ADR，哪些不值得？
2. 为什么 Accepted ADR 不应该被直接重写？
3. 技术债与 Bug 有什么区别？
4. 如何向业务负责人解释偿还技术债的价值？
5. 什么情况下应该接受技术债，而不是立即追求最佳方案？

## 十七、总结

- ADR 保存决策的 Context、Alternatives、Consequences 和状态。
- 高质量 ADR 从问题和 Decision Driver 开始，而不是从技术偏好开始。
- 技术债可以是有意识的商业选择，但必须透明、可量化且有 Owner。
- 债务排序应考虑业务影响、风险、持续利息和偿还成本。
- ADR、自动化 Fitness Function 和技术债治理共同支持架构长期演进。
- 架构治理的目标不是消除所有债务，而是确保团队理解并主动管理风险。
