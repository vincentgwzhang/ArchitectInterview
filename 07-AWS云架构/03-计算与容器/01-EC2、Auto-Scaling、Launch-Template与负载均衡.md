# 01. EC2、Auto Scaling、Launch Template 与负载均衡

## EC2

选择 Instance Family 依据 CPU、Memory、Network、Storage 和 Architecture。使用 Graviton 前验证 Java/Native Dependency。生产实例跨 AZ、无状态，数据放外部存储。

## Launch Template

版本化 AMI、Instance Type、Security Group、IAM Role、User Data 和 Metadata 配置。使用 IMDSv2、加密 EBS、最小 Role。AMI 采用 Packer/Image Pipeline 定期重建。

## Auto Scaling Group

设置 Min/Desired/Max、Multi-AZ、Health Check、Warmup、Termination Policy。Scaling Metric 可用 CPU、ALB Request Count、自定义 Queue；扩容后下游数据库也要有容量。

## Load Balancer

- ALB：HTTP/HTTPS、Path/Host Routing。
- NLB：高性能 TCP/UDP/TLS、Static IP。
- GWLB：Virtual Appliance 流量插入。

Readiness、Deregistration Delay 和 Graceful Shutdown 决定零停机。

## 购买与弹性

ASG 可混合 On-demand/Spot，关键 Base Capacity 保留稳定实例。Instance Refresh/Canary AMI 降低升级风险。

## 总结

EC2 架构不只是选机器。Immutable Launch Template、Multi-AZ ASG、正确 LB 与下游容量共同提供弹性。
