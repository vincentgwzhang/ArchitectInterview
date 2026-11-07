# 01. CloudWatch Metrics、Logs、Alarms 与 Dashboard

## Metrics

AWS 服务提供 Namespace/Dimension；应用发布 Custom Metric。使用 RED/USE 和业务指标，注意 High-resolution 与 Custom Metric 成本。避免 User ID 等高 Cardinality Dimension。

## Logs

CloudWatch Logs 使用 Log Group/Stream，设置 Retention、KMS、Subscription 与访问控制。应用输出 Structured JSON，禁止 Secret/PII。Logs Insights 用于临时查询，不应替代长期 Data Lake/SIEM 规划。

## Alarms

Static、Anomaly Detection、Composite Alarm。告警应对应用户影响或明确行动，配置 Missing Data 语义。SLO Burn Rate 比单个 CPU 阈值更适合 Page。

## Dashboard

按 User Journey 展示 Traffic、Error、Latency、Saturation、Deployment、Dependency 和 Cost。用变量/分区查看 Region、AZ、Version 和 Tenant Tier。

## Agent 与统一采集

CloudWatch Agent 收集 OS/Log；ADOT/OpenTelemetry 收集 Trace/Metric。统一 Resource Attribute 便于关联。

## 成本

控制 Log Volume/Retention、Metric Cardinality、Dashboard/Query 和 Data Transfer。采集策略按 Service Tier，不是所有 Debug Log 永久保存。

## 总结

CloudWatch 提供 AWS 原生信号，关键在统一语义、可行动告警、Retention 与成本治理，而不是默认开启所有数据。
