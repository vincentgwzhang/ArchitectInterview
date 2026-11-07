# 03. 可用性、容错、灾难恢复、RTO 与 RPO

## 学习目标

1. 区分 High Availability、Fault Tolerance 与 Disaster Recovery。
2. 理解 RTO、RPO、MTTR、MTBF 等指标。
3. 设计跨 Instance、AZ 和 Region 的故障策略。
4. 建立可测试的 Backup、Restore 和 DR Plan。

## 一、核心概念

### 1.1 High Availability（HA）

通过冗余、Failover 和自动恢复减少服务中断。通常关注常见故障，如 Instance 或 AZ 故障。

### 1.2 Fault Tolerance

系统在组件故障时继续提供服务，用户可能几乎无感。它通常比普通 HA 成本更高。

### 1.3 Disaster Recovery（DR）

针对 Region Failure、重大数据损坏、Security Incident 或人为误操作等灾难恢复业务能力。

### 1.4 Resilience

系统吸收故障、降级运行、恢复并从事故中改进的综合能力。

## 二、可靠性指标

- MTBF：Mean Time Between Failures。
- MTTD：Mean Time To Detect。
- MTTA：Mean Time To Acknowledge。
- MTTR：Mean Time To Restore/Repair。
- RTO：灾难发生后允许多长时间恢复服务。
- RPO：最多允许丢失多长时间的数据。

示例：

```text
RTO = 1 hour
RPO = 5 minutes
```

表示灾难后 1 小时内恢复，恢复点最多落后故障时刻 5 分钟。

RTO/RPO 越接近零，架构与运营成本越高，必须由 Business Impact Analysis 驱动。

## 三、Failure Domain

设计冗余前必须理解故障域：

- Process。
- Host/VM。
- Rack。
- Availability Zone。
- Region。
- Cloud Provider。
- Identity、DNS、CI/CD 等 Control Plane。

把三个 Replica 放在同一 Host 或同一 AZ，无法抵御该故障域失效。

还要避免 Correlated Failure：所有副本使用同一错误配置、同一过期证书或同一有缺陷版本时，数量冗余没有意义。

## 四、应用层高可用

典型设计：

- Service 无状态化。
- 跨至少两个 AZ 部署。
- Load Balancer 健康检查。
- Auto Scaling 与自动替换。
- Readiness/Liveness 区分。
- Graceful Shutdown。
- Rolling/Canary Deployment。
- Timeout、Circuit Breaker 和 Bulkhead。

Health Check 必须反映实例是否能安全接收请求。过浅会把请求发送到不可用实例，过深则可能因共享依赖故障同时摘除所有实例。

## 五、数据层高可用

需要考虑：

- Synchronous/Asynchronous Replication。
- Automatic Failover。
- Split Brain 防护。
- Backup 与 Point-in-time Recovery。
- Data Corruption Detection。
- Failover 后 Client Connection 行为。

Replica 不是 Backup。错误删除或逻辑损坏可能立即复制到所有 Replica。Backup 必须独立、受保护并经过恢复验证。

## 六、DR 策略

### 6.1 Backup and Restore

成本最低，RTO/RPO 较大。灾难后创建 Infrastructure 并恢复数据。

### 6.2 Pilot Light

备用 Region 保留核心数据和最小关键组件，灾难时扩容应用层。

### 6.3 Warm Standby

备用 Region 运行缩小版完整系统，可在灾难时扩容并切流。

### 6.4 Multi-site Active-Active

多个 Region 同时承载流量，RTO 较低，但需要解决 Global Routing、Data Consistency、Conflict 和更高成本。

| 策略 | 成本 | 典型 RTO | 复杂度 |
| --- | --- | --- | --- |
| Backup/Restore | 低 | 小时到天 | 低 |
| Pilot Light | 中低 | 小时 | 中 |
| Warm Standby | 中高 | 分钟到小时 | 中高 |
| Active-Active | 高 | 秒到分钟 | 高 |

具体时间必须通过演练验证，不能只根据架构图声明。

## 七、Backup 设计

Backup Plan 需要定义：

- 数据范围。
- Full/Incremental/PITR 策略。
- Frequency 与 Retention。
- Encryption 和 Key Access。
- Cross-account / Cross-region Copy。
- Immutability 和删除保护。
- Restore Procedure。
- Restore Test Frequency。

遵循 3-2-1 思路：至少三份副本、两种介质或故障域、一份异地。Cloud 环境中可映射为生产数据、独立 Backup Vault 和跨 Account/Region Copy。

## 八、Failover 与 Failback

Failover 不是终点。还需要设计：

- 谁有权宣布 Disaster？
- DNS TTL 和流量切换时间。
- 数据复制是否已追平？
- 是否需要 Fence 旧 Primary 防止双写？
- Client 如何重新连接？
- 原 Region 恢复后如何 Failback？
- Failback 期间数据如何同步？

自动 Failover 很快，但错误判断可能扩大事故；人工 Failover 更谨慎，却增加 RTO。可根据故障类型采用不同策略。

## 九、Graceful Degradation

当完整服务无法维持时，可以优先保留核心能力：

- 推荐不可用时展示热门商品。
- Search 不可用时提供 Category 浏览。
- Analytics Event 暂存 Queue。
- 只读模式保护数据。
- 限制大客户或高成本 Query。

需要提前定义 Degraded Mode，并定期测试。事故中临时发明降级方案风险很高。

## 十、DR 演练

Paper Plan 不能证明系统可以恢复。演练应验证：

1. Alarm 能否发现故障。
2. On-call 是否能找到正确 Runbook。
3. 权限和凭证是否可用。
4. Infrastructure 能否重建。
5. Backup 是否能恢复。
6. DNS/Traffic 能否切换。
7. 数据是否一致完整。
8. 实际 RTO/RPO 是否达标。

Game Day 后应记录 Gap、Owner 和截止日期。

## 十一、常见错误

- 把 Multi-AZ 当成完整 DR。
- 把 Replica 当 Backup。
- 有 Backup 但从未 Restore Test。
- 所有环境依赖同一 AWS Account 或 IAM Control Plane。
- 只设计 Failover，不设计 Failback。
- RTO/RPO 由技术团队猜测，没有业务确认。
- Active-Active 只有流量方案，没有数据冲突方案。

## 十二、面试回答框架

```text
1. 根据业务影响确定 Tier 和 RTO/RPO。
2. 列出 Process、AZ、Region、Data Corruption 等故障场景。
3. 为常见故障设计自动恢复和 Multi-AZ。
4. 为 Region Disaster 选择合适 DR Strategy。
5. 设计 Backup、Replication、Traffic Shift 和 Failback。
6. 通过 Monitoring、Runbook 和定期 Game Day 验证。
7. 说明 Cost 与 Residual Risk。
```

## 十三、自测题

1. HA 与 DR 有什么区别？
2. RTO 为 1 小时、RPO 为 5 分钟分别表示什么？
3. 为什么 Replica 不能替代 Backup？
4. Warm Standby 与 Active-Active 的核心差异是什么？
5. 为什么 DR Plan 必须包含 Failback？

## 十四、总结

- Availability、Fault Tolerance 和 DR 解决不同等级的故障。
- RTO/RPO 必须来自业务影响，并通过演练证明。
- 冗余必须跨越目标 Failure Domain，并防止 Correlated Failure。
- Backup、Restore、Failover 和 Failback 都是完整恢复能力的一部分。
- 高可靠性来自架构、自动化、运营纪律和持续演练，而不只是增加副本。
