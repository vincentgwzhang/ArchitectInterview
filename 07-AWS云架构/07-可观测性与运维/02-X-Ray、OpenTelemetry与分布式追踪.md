# 02. X-Ray、OpenTelemetry 与分布式追踪

## X-Ray

AWS 分布式追踪服务，收集 Segment/Subsegment、Service Map、Error 与 Latency。与 API Gateway、Lambda 等集成。

## OpenTelemetry

Vendor-neutral API/SDK/Collector，支持 Trace、Metric、Log。AWS Distro for OpenTelemetry（ADOT）可向 X-Ray/CloudWatch 或其他 Backend 导出。

## Context Propagation

HTTP、gRPC、SQS/SNS/Kafka 传播 Trace/Causation。不要信任外部 Client 传入的任意内部 Attribute；生成/验证 Trace Context。

## Instrumentation

Auto Instrumentation 快速覆盖 Framework，Manual Span 记录关键业务步骤。避免在 Span 放 PII、Token 和巨大 Payload。

## Sampling

Head Sampling 在请求开始决定，成本低但可能漏掉后续错误；Tail Sampling 可根据完整 Trace 保留 Error/Slow，需 Collector Buffer。按关键服务和成本组合。

## 关联

Metric Exemplar → Trace，Trace ID → Structured Log；统一 service.name、version、environment、region。

## 总结

X-Ray 提供 AWS Backend，OpenTelemetry 提供统一采集标准。高质量 Context、Sampling 和业务 Span 决定诊断价值。
