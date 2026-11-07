# 04. AWS 部署、可观测性、安全与成本控制

## 部署

CloudFront/WAF → ALB/API Gateway → ECS/EKS/Lambda；Aurora/DynamoDB 保存会话与元数据；S3 文档；OpenSearch/Vector Store 检索；SQS/Step Functions 接入任务。跨 AZ，无状态计算。

## 安全

Multi-account、Private Subnet/Endpoint、IAM Role、KMS、Secrets Manager、CloudTrail/GuardDuty。Provider 选择符合数据驻留与 Retention。Prompt/Response Log 默认 Redact。

## Observability

Trace 覆盖 API、Retrieval、Model、Tool；记录 Prompt/Model/Index Version、Token、TTFT、Citation、ACL Decision。质量指标与技术指标分层。

## 成本

Cost per Answer/Successful Task；限制 Token、并发和每日 Tenant Budget；Batch Embedding、Cache、Model Routing、Storage Lifecycle。NAT/OpenSearch/Log 常是隐性成本。

## DR

S3 Version/Replication、Database Backup、Index 可从 Source 重建、Prompt/Config/IaC 跨 Region。按业务 RTO/RPO 选择 Warm Standby 或 Restore。

## 总结

生产 AI 平台需同时治理数据、模型、基础设施与单位成本。Managed Service 减少运维，但不转移安全和质量责任。
