# 03. 存储生命周期、数据传输与 NAT 成本优化

## 存储

按 Access Pattern 使用 S3 Lifecycle/Intelligent-Tiering、EBS gp3、Snapshot Retention、OpenSearch/Log 冷热层。删除无业务/合规价值的数据通常比换存储级别收益更大。

注意最短存储期、Retrieval、Request 和 Early Deletion Fee。

## 数据传输

绘制 Data Flow：Internet Egress、Cross-AZ、Cross-region、NAT、CloudFront Origin。架构图应标记流量与每 GB 费用。

优化：

- CDN Cache/Compression。
- Compute 与 Data 同 Region/AZ（兼顾 HA）。
- 减少重复 Payload。
- Batch/压缩 Event。
- 避免无意跨 AZ Hairpin。

## NAT

NAT Gateway 按小时和处理量收费。S3/DynamoDB Gateway Endpoint、Interface Endpoint（需比较自身费用）、本地 Registry Mirror 和合理 Egress 可降低 NAT。

每 AZ NAT 可避免 Cross-AZ 并提高 HA，但小时费增加；按流量和风险计算。

## 可观测性

CUR Usage Type、VPC Flow Logs、NAT Metric、CloudFront Hit Ratio 和 S3 Storage Lens 定位成本。

## 总结

存储与网络成本来自数据生命周期和拓扑。先理解字节从哪里到哪里，再选择分层、Endpoint 和 Cache。
