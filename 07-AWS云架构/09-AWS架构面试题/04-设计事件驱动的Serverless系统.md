# 04. 设计事件驱动的 Serverless 系统

## 场景

以文件处理为例：用户上传 S3，触发扫描、解析、转换和通知。要求峰值弹性、失败恢复和可追踪。

## 架构

```text
Client -> Pre-signed S3 Upload
S3 Event -> EventBridge/SQS -> Lambda Workers
-> Step Functions for orchestration
-> DynamoDB state / S3 result
-> SNS/WebSocket notification
```

## 可靠性

SQS Buffer、Visibility、DLQ；Lambda Reserved Concurrency 保护下游；Event ID/对象版本确保幂等；Step Functions 保存长流程和补偿。

## 安全

Pre-signed URL 限制 Key/大小/时效，上传后隔离扫描；每 Function 最小 IAM；KMS、VPC Endpoint、CloudTrail。不要信任文件扩展名。

## 可观测

Correlation ID/Object Version、Queue Age、Lambda Error/Throttle、Workflow Duration、DLQ 和 Business Completion。

## 成本

控制 Lambda Memory/Duration、S3 Request、Step Function Transition、NAT 和 Log。高持续吞吐时比较 ECS/Kinesis。

## 面试亮点

说明 At-least-once、幂等、Poison Event、Backpressure、配额和安全文件处理，而不是只列 Lambda 服务。
