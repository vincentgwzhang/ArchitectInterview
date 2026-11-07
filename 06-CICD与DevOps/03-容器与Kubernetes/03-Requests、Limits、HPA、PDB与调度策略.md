# 03. Requests、Limits、HPA、PDB 与调度策略

## Requests 与 Limits

Request 用于调度和 Guaranteed Capacity；Limit 限制最大使用。CPU 超 Limit 会 Throttle，Memory 超 Limit 通常 OOMKill。

通过生产 Metric/Load Test Rightsize。Request 过高浪费与阻塞调度，过低造成 Node Overcommit 与不稳定。

## QoS

Guaranteed、Burstable、BestEffort 影响资源压力下 Eviction。关键 Service 应有合理 Request/Limit，不使用 BestEffort。

## HPA

基于 CPU、Memory 或 Custom Metric 调整 Replica。对 API 可用 RPS/Latency，Queue Consumer 可用 Lag/Age。设置 Min/Max、Behavior、Warmup，并确保下游能承受扩容。

Memory 常不随流量快速下降，不一定是理想 Scaling Signal。

## PDB

限制 Voluntary Disruption 同时不可用数量，保护 Node Drain/Upgrade。它不防 Node Crash，也不能替代多个 Replica 与跨 AZ 调度。

## 调度

- Topology Spread/Anti-affinity 跨 Node/AZ。
- Node Affinity 选择硬件。
- Taint/Toleration 隔离 Workload。
- PriorityClass 处理资源紧张。

规则过强可能让 Pod Pending，必须平衡可用性与可调度性。

## 总结

资源和调度策略共同决定容量、成本与故障域。HPA 不是无限扩容，必须与 Request、启动速度和依赖容量联合设计。
