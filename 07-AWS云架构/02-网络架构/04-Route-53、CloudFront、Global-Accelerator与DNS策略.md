# 04. Route 53、CloudFront、Global Accelerator 与 DNS 策略

## Route 53

权威 DNS 与 Health Check。Routing Policy 包括 Simple、Weighted、Latency、Failover、Geolocation/Geoproximity 和 Multi-value。

DNS Failover 受 TTL、Resolver Cache 和 Health Check 影响，不是实时连接切换。变更前降低 TTL，但低 TTL 增加 Query 与成本。

## CloudFront

CDN 缓存 HTTP 内容、TLS/WAF/Edge Logic，适合静态/动态 Web、API 加速和 Origin Protection。Cache Key 需控制 Header/Cookie/Query，否则 Hit Ratio 低或私有数据泄漏。

## Global Accelerator

提供 Anycast Static IP，通过 AWS Global Network 将 TCP/UDP 流量导向健康 Regional Endpoint，快速故障转移。它不是内容 Cache。

## 选择

- DNS 级地域/权重：Route 53。
- Web/CDN/Cache/Edge Security：CloudFront。
- 非 HTTP、Static IP、快速连接路由：Global Accelerator。

可组合使用。

## DNS 策略

Split-horizon Public/Private Hosted Zone、Alias Record、DNSSEC（适用时）、Domain/Certificate Renewal 和 Resolver Logging。避免 CNAME 链与循环。

## 总结

Route 53 做名称与流量策略，CloudFront 做边缘内容交付，Global Accelerator 做全球网络入口。Failover 必须实际演练。
