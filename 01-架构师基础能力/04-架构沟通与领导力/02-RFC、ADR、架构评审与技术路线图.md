# 02. RFC、ADR、架构评审与技术路线图

## 学习目标

1. 区分 RFC、ADR、Architecture Review 和 Technology Roadmap。
2. 设计轻量但有效的架构治理流程。
3. 让决策、执行和长期方向形成闭环。
4. 避免评审成为审批 Bottleneck。

## 一、四种工具的职责

### RFC：Request for Comments

在重大决策前提出问题、方案和开放问题，用于收集 Feedback 和形成共识。

### ADR：Architecture Decision Record

记录已经作出的重要决策、Context、Alternatives 和 Consequences。

### Architecture Review

由相关 Stakeholder 对方案的质量属性、风险和一致性进行结构化检查。

### Technology Roadmap

描述技术能力如何分阶段支持业务方向，包括时间、依赖、里程碑和投资。

可以理解为：

```text
RFC 探索和讨论
  -> Review 挑战和验证
  -> ADR 记录决定
  -> Roadmap 安排演进
  -> Metric/Feedback 验证效果
```

## 二、RFC 结构

一份实用 RFC 可以包含：

```text
Title / Status / Author / Reviewers
Context and Problem
Goals and Non-goals
Requirements and Constraints
Options Considered
Proposed Design
Data/API/Security/Operations
Trade-offs
Migration and Rollback
Open Questions
Decision Deadline
```

RFC 的目标不是写得最长，而是让异步参与者能够理解并提出具体反馈。

### RFC 何时需要

- 跨团队 Contract。
- 新 Platform 或基础技术。
- 高成本、难逆转决策。
- 重要 Data/Security 影响。
- 大规模 Migration。

局部、可逆、低风险决策不应被迫走重流程。

## 三、RFC Review

明确角色：

- Author：准备方案并处理 Feedback。
- Reviewer：从相关领域提出问题。
- Decider：在无法完全共识时做最终决定。
- Consulted：提供 Security、SRE、Data 等专业意见。
- Informed：需要知道结果但无需批准。

Feedback 应标注：

- Blocking：必须解决才可接受。
- Important：强烈建议解决。
- Suggestion：非阻塞改进。
- Question：需要澄清。

“我不喜欢”不是有效 Blocking Feedback。Blocking Concern 应关联 Requirement、Risk、Standard 或 Evidence。

## 四、ADR 与 RFC 的关系

RFC 可以很长并包含讨论历史；ADR 应简洁记录最终决定。

例如：

```text
RFC-0012: Evaluate messaging platform
ADR-0028: Adopt Amazon MSK for domain event streaming
```

ADR 链接 RFC，保留 Alternatives 和 Proof of Concept 结果。后续如果切换到其他方案，新建 ADR 并标记旧 ADR 为 Superseded。

## 五、Architecture Review

评审重点不应是命名和个人偏好，而应系统检查：

- 是否满足 Functional Requirement 和 NFR？
- Capacity Assumption 是否合理？
- Data Ownership 和 Consistency 是否明确？
- Security/Privacy/Compliance 是否覆盖？
- Failure Mode、SLO、DR 是否完整？
- Operability、Migration、Rollback 是否可行？
- Cost 和 Team Capability 是否匹配？
- 是否存在更简单方案？

可以按风险决定评审深度：

| Risk | 流程 |
| --- | --- |
| Low | Team 自主决策，轻量 ADR |
| Medium | Domain Architect/Peer Review |
| High | 跨职能 Review + Prototype + 正式 Sign-off |

## 六、避免架构评审 Bottleneck

有效方法：

- 发布 Reference Architecture 和 Golden Path。
- 使用 Self-service Checklist。
- 明确哪些情况必须升级评审。
- 设定 Review SLA。
- 支持异步 Review。
- 自动检查可编码规则。
- 记录 Exception 与到期时间。

架构团队应提供 Enablement，而不是成为所有变更的中央审批者。

## 七、技术路线图

Technology Roadmap 不应是工具 Wishlist。它应连接：

```text
Business Objective
    -> Required Capability
        -> Architecture Initiative
            -> Milestone and Metric
```

示例：

```text
Business objective: 进入欧洲企业市场
Capability: EU data residency + 99.95% SLA
Initiatives:
- EU Landing Zone
- Tenant-aware data placement
- Cross-region release pipeline
- Audit and retention controls
Milestones:
- Q1 foundation
- Q2 pilot tenants
- Q3 general availability
Metrics:
- provisioning < 1 day
- RTO < 1 hour
- compliance controls 100% automated evidence
```

## 八、Roadmap 层级

可以分为：

- Now：已承诺、范围清晰。
- Next：较高置信度，仍需验证。
- Later：方向性，取决于 Trigger。

避免为两年后的具体技术和日期制造虚假确定性。越远期越应描述 Capability 和 Outcome，而不是精确 Tool。

## 九、管理依赖与风险

Roadmap 应标出：

- Team Dependency。
- Vendor/Contract Dependency。
- Data Migration。
- Compliance Deadline。
- Hiring/Skill Gap。
- Critical Path。
- Risk Reduction Milestone。

架构 Initiative 可以先做 Enabler，但必须说明它支持哪个业务结果，避免长期“打地基”却没有可见价值。

## 十、治理闭环

```text
Standard / Roadmap
  -> RFC / Design
  -> Review / Decision
  -> ADR
  -> Implementation
  -> Fitness Function / SLO / Cost Metric
  -> Feedback and Roadmap Update
```

如果只评审设计而不验证结果，治理是不完整的。

需要检查：

- 方案上线后是否达到 SLO？
- 真实成本是否符合预测？
- Incident 是否暴露新风险？
- ADR Assumption 是否仍成立？
- Roadmap 是否需要调整？

## 十一、常见错误

- RFC 写完后没有 Decider 和 Deadline。
- ADR 只写“选择了什么”，不写为什么。
- 所有改动都经过中央委员会。
- Review 过度关注 Style，忽略 Failure 和 Migration。
- Roadmap 只列 Technology，不连接 Business Outcome。
- 长期路线图过度精确，从不更新。
- 决策完成后没有验证 Metric。

## 十二、面试回答框架

> 我会按 Decision Risk 调整治理强度。跨团队、高成本或涉及敏感数据的变更先通过 RFC 异步收集意见，再进行以 NFR、Failure、Security 和 Migration 为核心的 Review。最终决定写入 ADR。长期能力进入与业务目标绑定的 Roadmap，并通过 SLO、Cost 和 Fitness Function 验证。低风险决策由团队自主完成，以免架构团队成为 Bottleneck。

## 十三、自测题

1. RFC 与 ADR 的核心区别是什么？
2. 哪类决策需要正式 Architecture Review？
3. 如何避免 Review 流程降低交付速度？
4. Technology Roadmap 为什么不应该只是 Tool List？
5. 架构治理如何形成 Feedback Loop？

## 十四、总结

- RFC 用于决策前讨论，ADR 用于决策后留痕。
- Review 应聚焦质量属性、风险和可执行性。
- 治理强度应与决策风险匹配。
- Roadmap 将业务目标映射为 Capability、Initiative 和可量化里程碑。
- 架构治理只有在实现结果被度量并反馈时才真正闭环。
