# 05. WAF、Shield、Network Firewall 与分层防护

## AWS WAF

Layer 7 Web Request 过滤，可保护 CloudFront、ALB、API Gateway。规则包括 Managed Rule、IP、Rate-based、Geo、Header/Body Pattern。先 Count Mode 观察 False Positive，再 Block。

## Shield

Shield Standard 提供基础 DDoS 防护；Shield Advanced 面向高风险应用，提供增强检测、响应与成本保护等能力。DDoS 设计仍需 Auto Scaling、CloudFront/Global Accelerator 和 Runbook。

## Network Firewall

VPC 级 Stateful Network Inspection，适合集中 Egress/Ingress、Domain/Protocol Control。需要规划 Routing、Multi-AZ、Asymmetric Flow、吞吐和成本。

## 分层

```text
Route53/Shield -> CloudFront/WAF -> ALB/SG
-> App Authorization -> Data IAM/Encryption
```

网络控制不能替代应用 Authorization、Input Validation 和 Rate Limit。

## 运营

记录 WAF/Firewall Log 到受控位置，建立 Attack/False Block Metric。规则更新 Canary，Emergency Block 有审批和到期时间。

## 总结

Shield 抵御 DDoS，WAF 过滤 Web 攻击，Network Firewall 管网络流量。Defense in Depth 必须连接身份、应用和数据控制。
