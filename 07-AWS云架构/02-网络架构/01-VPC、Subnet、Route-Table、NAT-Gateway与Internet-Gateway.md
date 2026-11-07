# 01. VPC、Subnet、Route Table、NAT Gateway 与 Internet Gateway

## VPC 与 Subnet

VPC 是 Region 级逻辑网络，Subnet 属于单一 AZ。CIDR 规划要为增长、Peering/TGW 和 On-premise 避免重叠，并保留地址空间。

Public Subnet 的 Route 指向 Internet Gateway（IGW），但实例仍需 Public IP 和 Security Rule；Private Subnet 通常通过 NAT Gateway 出站。

## Route Table

Longest Prefix Match 决定路径。每个 Subnet 关联 Route Table。使用 Flow Logs、Reachability Analyzer 和 Route 检查排查连接。

## NAT Gateway

托管、AZ 级出站 NAT。为高可用和避免跨 AZ 流量，通常每 AZ 一个 NAT，并让 Private Subnet 使用本 AZ NAT。NAT 有处理费与数据费，是常见成本热点。

对 S3/DynamoDB 使用 Gateway Endpoint，对其他 AWS 服务使用 Interface Endpoint，可减少 NAT、提高私有访问。

## 分层

典型：Public Subnet 放 ALB/NAT，Private App Subnet 放计算，Isolated Data Subnet 放数据库。不是所有工作都需要三层，但 Trust Boundary 要明确。

## 安全

Route 只决定可达性，不等于授权；Security Group/NACL/IAM/应用权限共同控制。

## 总结

VPC 设计从 CIDR、故障域、流量路径与成本开始。Public/Private 由路由和地址共同决定，不是命名决定。
