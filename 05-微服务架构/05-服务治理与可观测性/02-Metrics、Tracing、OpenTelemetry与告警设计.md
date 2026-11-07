# 02. Metrics、Tracing、OpenTelemetry 与告警设计

## Metrics

Service 使用 RED：Rate、Error、Duration；Resource 使用 USE：Utilization、Saturation、Error。Histogram Bucket 应匹配 SLO，避免无法计算 P95/P99。

控制 Label Cardinality，不使用 Request/User/Order ID。

## Tracing

Span 展示跨 Service、DB、Broker 的 Critical Path。传播 W3C Trace Context，异步消息使用 Link/Causation 表达关系。Sampling 保留 Error/Slow/关键交易。

## OpenTelemetry

统一 API/SDK、Auto Instrumentation、Collector 和 Exporter，降低 Vendor Lock-in。仍需定义 Semantic Convention、Resource Attribute、Sampling 和敏感数据策略。

## 告警

优先用户影响：SLO Burn Rate、Error、Latency、Queue Age、业务失败。基础资源 Alert 用于诊断或预测 Saturation。

每个 Page Alert 必须有 Owner、Severity、Runbook、Dashboard 和明确动作。无行动告警应删除或降级。

## Trace-Metric-Log

Exemplar 从 Metric 跳到 Trace，Trace ID 从 Trace 跳 Log。统一 Service Name、Version、Region 和 Deployment 标识。

## 总结

Metrics 发现问题，Trace 定位路径，Log 提供细节。OpenTelemetry 建立统一采集，SLO-based Alert 保持可行动性。
