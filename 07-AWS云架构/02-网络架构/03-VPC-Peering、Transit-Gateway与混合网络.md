# 03. VPC Peering、Transit Gateway 与混合网络

## VPC Peering

两个 VPC 点对点连接，低延迟且简单，但不支持 Transitive Routing。VPC 数量增长后形成 Full Mesh Route/管理复杂度，CIDR 不能重叠。

## Transit Gateway（TGW）

Hub-and-spoke 连接多个 VPC、VPN 和 Direct Connect Gateway。通过 TGW Route Table 做环境/域隔离和集中 Inspection。付 Attachment 与流量费用。

## 混合连接

- Site-to-Site VPN：通过 Internet 加密，部署快，可作备份。
- Direct Connect：专线、稳定吞吐，非自动加密，需冗余 Location/Connection。
- DX + VPN：兼顾专线与加密/备份。

## DNS

使用 Route 53 Resolver Inbound/Outbound Endpoint 和 Rule 实现 On-premise/AWS 双向解析。DNS 设计常是混合网络成败关键。

## 路由与安全

明确 CIDR、BGP、Route Propagation、Asymmetric Routing、Inspection Path 和 Failure。集中 Firewall 会增加成本、延迟和故障域。

## 选择

少量 VPC 点对点可 Peering；多账户规模化、集中网络和 Hybrid 通常 TGW。重叠 CIDR 可使用 PrivateLink、Proxy 或重新编号。

## 总结

网络拓扑随 VPC 数量演进。Peering 简单，TGW 可治理，Hybrid 还需 DNS、冗余和端到端路由验证。
