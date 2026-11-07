# 01. Region、Availability Zone 与 Edge Location

AWS Region 是独立地理区域；一个 Region 包含多个相互隔离但低延迟连接的 Availability Zone（AZ）。Edge Location 用于 CloudFront、Route 53 等边缘服务，就近服务用户，不等同于完整 Region。

## 设计原则

- 生产应用至少跨两个 AZ，关键系统通常跨三个。
- Region 选择考虑用户延迟、服务可用性、价格、合规和数据驻留。
- Multi-AZ 处理常见设施故障；Multi-region 才针对 Region Disaster。
- AZ 名称在不同 Account 中不一定映射相同物理 AZ，可使用 AZ ID 协调。

## 故障域

把多个实例放进同一 AZ 不构成 AZ Resilience。Application、Load Balancer、NAT、Database 和消息系统都要检查实际分布。AZ 故障后剩余容量必须承载 Peak，而不仅是“还有实例”。

Edge Cache 可以降低读取延迟和 Origin 压力，但写入、个性化与强一致数据仍需回 Region。CloudFront Origin Failover 也不能自动解决数据库跨 Region 恢复。

## 面试要点

解释故障域、数据复制、流量路由和成本，而不是只说“部署多个 Region”。更高冗余必须对应 RTO/RPO 与业务价值。

## 总结

Region 决定地理与合规边界，AZ 是区域内高可用单元，Edge 优化全球访问。三者承担不同职责。
