# 04. Build vs Buy、技术选型与供应商评估

## 学习目标

1. 使用业务与工程标准进行 Build vs Buy 决策。
2. 计算 Total Cost of Ownership，而不只比较采购价格。
3. 系统评估 Vendor 的技术、安全、商业与运营风险。
4. 设计可执行的 Proof of Concept、Contract 和 Exit Strategy。

## 一、Build vs Buy 不是二元选择

常见选择包括：

- Build：完全自研。
- Buy SaaS：购买完整能力。
- Managed Service：使用托管基础服务。
- Open Source Self-hosted。
- Open Core / Commercial Distribution。
- Hybrid：核心能力自研，通用能力购买。

真正的问题是：哪些能力构成业务差异化，哪些能力更适合复用市场成熟方案。

## 二、Decision Driver

### 2.1 Strategic Differentiation

如果能力直接形成竞争优势、独特数据资产或核心 Customer Experience，自研价值更高。

Payroll、Email Delivery、Commodity Monitoring 等通用能力通常不构成差异化，更适合 Buy 或 Managed Service。

### 2.2 Time to Market

Buy 往往能加速初期交付，但 Integration、Migration、Security Review 和 Procurement 仍需要时间。

### 2.3 Control 与 Customization

Build 提供更高控制，但团队承担全部 Roadmap、Reliability、Security 和 Support。

### 2.4 Scale 与 Economics

小规模时 SaaS 通常经济；规模扩大后按 Usage 定价可能昂贵。自研也有高固定成本和持续维护成本。

### 2.5 Capability 与 Skill

团队是否具备开发、运行和 24×7 Support 能力？一个 Prototype 能运行，不代表它适合生产。

## 三、Total Cost of Ownership

TCO 应包含：

```text
Build TCO = Development
          + Infrastructure
          + Operations/On-call
          + Security/Compliance
          + Upgrade/Maintenance
          + Opportunity Cost

Buy TCO = Subscription/Usage
        + Integration
        + Migration
        + Vendor Management
        + Training
        + Exit/Switching Cost
```

至少比较 3 年或符合产品生命周期的周期，并做增长与价格敏感性分析。

## 四、技术选型评分维度

| 维度 | 评估问题 |
| --- | --- |
| Functional Fit | 是否满足核心 Use Case？ |
| NFR | Latency、Scale、Availability、DR 如何？ |
| Security | IAM、Encryption、Audit、Vulnerability Process？ |
| Compliance | Data Residency、Certification、DPA？ |
| Integration | API、SDK、Event、SSO、Data Export？ |
| Operability | Monitoring、Support、Upgrade、Incident？ |
| Economics | 3 年 TCO、价格增长、Egress？ |
| Vendor Risk | 财务稳定性、Roadmap、Acquisition Risk？ |
| Lock-in | 数据/流程迁移难度、标准兼容性？ |
| Team Fit | 技能、招聘、学习曲线？ |

先定义权重，再评分。不要先喜欢某个工具，然后设计评分标准证明它最好。

## 五、供应商 Due Diligence

### 5.1 技术

- Architecture 与 Scaling Model。
- API Limit、Quota 和 Rate Limit。
- SLA 与历史可靠性。
- Backup、RTO、RPO。
- Versioning 和 Deprecation Policy。
- Observability 与 Export 能力。

### 5.2 Security 与 Privacy

- SSO、SCIM、RBAC/ABAC。
- Encryption 与 Key Option。
- Audit Log。
- Tenant Isolation。
- Secure SDLC 与 Pen Test。
- Incident Notification。
- Subprocessor 与 Data Location。
- Data Deletion。

### 5.3 商业与法律

- Pricing Metric 和 Overage。
- Minimum Commitment。
- Renewal Cap。
- Support Tier。
- Liability 与 Indemnity。
- Data Ownership。
- Termination Assistance。
- Vendor Acquisition/Bankruptcy 条款。

## 六、Proof of Concept

有效 PoC 应验证最大风险，而不是演示 Happy Path。

例如评估 Vector Database：

```text
Dataset: 10 million vectors
Query mix: production-like
Success criteria:
- Recall@10 >= 0.92
- P95 latency < 150 ms
- ingestion >= 20k vectors/sec
- tenant filtering correctness = 100%
- projected monthly cost < €X
- backup restore < 2 hours
```

同时测试：

- Failure 和 Recovery。
- Security Integration。
- Data Export。
- Upgrade/Schema Change。
- Operational Complexity。

PoC 要有 Timebox，避免无期限探索。

## 七、Lock-in 的理性分析

Vendor Lock-in 不是绝对坏事。使用高层 Managed Service 可以显著缩短交付并减少运维。

应该评估：

- 退出概率。
- 退出影响。
- 替代方案。
- 数据迁移时间。
- Contract/API 专有程度。
- 为可移植性支付的当前成本。

降低 Lock-in 的方法：

- 保留数据 Export。
- 在业务层隔离 Vendor SDK。
- 使用开放 Protocol 和 Format。
- 避免关键业务规则只存在于 Vendor 配置。
- 定期测试 Restore/Export。
- Contract 中约定 Termination Support。

不要为理论上的完全可移植性建立复杂抽象，却从未测试真实迁移。

## 八、Exit Strategy

在签约前回答：

- 如何导出全部数据？格式是什么？
- 导出需要多久、费用多少？
- 历史 Audit Log 能否导出？
- 删除后如何获得证明？
- 替代系统并行期如何同步数据？
- 哪些 Application 依赖专有 API？
- Contract 终止后有多长迁移窗口？

Exit Plan 不意味着准备立即退出，而是控制不可逆风险。

## 九、场景：Authentication Platform

Build 的优势：完全控制流程与数据。  
Build 的风险：Password Security、MFA、Account Recovery、Attack Detection、Compliance 和 24×7 Support。

Buy 的优势：快速获得成熟安全能力。  
Buy 的风险：按 MAU 定价、Login Critical Dependency、Customization 和迁移成本。

对于多数非身份核心业务，购买成熟 Identity Provider 更合理，但仍应设计：

- Local User Mapping。
- Token Validation Resilience。
- Provider Outage 行为。
- Data Export。
- Break-glass Access。
- Contract SLA。

## 十、决策与复审

最终 ADR 应记录：

- Driver 和权重。
- 候选方案。
- PoC Evidence。
- TCO Assumption。
- Security/Legal 结论。
- Risk 与 Mitigation。
- Contract 关键条件。
- Exit Strategy。
- Revisit Trigger。

典型 Trigger：使用量增长、价格变化、SLA 失败、产品 Roadmap 偏离或新合规要求。

## 十一、常见错误

- 只比较 License Price。
- 把 Open Source 当作“免费”。
- PoC 只验证 Demo，不验证 Scale 和 Failure。
- 忽略 Procurement、Legal 和 Security Lead Time。
- 为避免 Lock-in 拒绝所有 Managed Service。
- 没有数据导出和退出条款。
- 供应商选择依赖个人关系或品牌，而不是 Evidence。

## 十二、自测题

1. 哪些能力更适合 Build，哪些更适合 Buy？
2. TCO 中最容易遗漏哪些成本？
3. PoC 为什么应该优先验证最大风险？
4. 如何量化 Vendor Lock-in？
5. Exit Strategy 应在何时设计？

## 十三、总结

- Build vs Buy 应从差异化、Time to Market、TCO、风险和团队能力出发。
- Vendor Evaluation 同时覆盖技术、安全、合规、商业和运营。
- PoC 必须使用生产式数据和可量化成功标准验证风险。
- Lock-in 应理性管理，而不是追求不现实的零依赖。
- Contract、Data Export、Exit Plan 和 Revisit Trigger 是选型的一部分。
