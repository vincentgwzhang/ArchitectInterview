# 02. Security Group、NACL、PrivateLink 与 VPC Endpoint

## Security Group

Stateful Virtual Firewall，绑定 ENI/Resource，只需定义允许规则，返回流量自动允许。可引用另一个 SG，适合表达 ALB → App → DB，而不是维护 IP。

## NACL

Stateless、Subnet 级，按 Rule Number 顺序处理 Allow/Deny；返回流量也需显式允许 Ephemeral Port。适合粗粒度 Boundary 或明确 Deny，但复杂 NACL 易造成故障。

## VPC Endpoint

- Gateway Endpoint：S3/DynamoDB，更新 Route Table，无小时费。
- Interface Endpoint：基于 PrivateLink，在 Subnet 创建 ENI，使用 SG，有小时/流量成本。

Endpoint Policy 与 Service IAM Policy 共同限制访问。

## PrivateLink

跨 VPC/Account 私有暴露服务，不需要路由整个网络，Consumer 通过 Interface Endpoint 连接 Provider 的 Endpoint Service/NLB。适合 SaaS/共享服务和重叠 CIDR。

## 设计

默认 Deny 思维、最小 Port/Source、Flow Logs、集中 DNS。注意 Interface Endpoint 每 AZ 成本和 DNS Resolution。

## 总结

SG 是主要实例级控制，NACL 是无状态 Subnet Guardrail，Endpoint/PrivateLink 让服务私网可达并减少公网/NAT 依赖。
