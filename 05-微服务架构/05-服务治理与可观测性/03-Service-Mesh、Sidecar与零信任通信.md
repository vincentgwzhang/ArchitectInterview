# 03. Service Mesh、Sidecar 与零信任通信

## Service Mesh

通过 Data Plane Proxy 和 Control Plane 提供 mTLS、Traffic Policy、Telemetry、Retry 和 Service Identity。Sidecar 每 Pod 部署代理；也有 Ambient/Node-level 模式。

## 价值

- 一致的 Workload Identity/mTLS。
- 细粒度 Traffic Split/Canary。
- 语言无关 Telemetry。
- 统一 Timeout/Policy。

## 成本

额外 Latency、CPU/Memory、配置复杂度、Control Plane 运维、双代理 Debug 和升级风险。小规模系统可能不值得。

## Zero Trust

不因“在内网”自动信任。每个 Workload 有身份，通信加密并按 Service/Operation 最小授权。Network Policy 限制路径，Application 仍验证 End-user Authorization。

## Retry 风险

Mesh 默认 Retry 可能与应用 Retry 叠加，且不了解业务幂等。明确每层 Policy，写操作谨慎。

## Sidecar 生命周期

处理 Startup/Shutdown 顺序、Job 完成、Port Capture、Probe 和证书轮换。Proxy 未 Ready 时应用不应接收流量。

## 采用条件

当服务/语言多、mTLS 和 Traffic Governance 需求明确、Platform Team 能运营时评估。先用小范围 Use Case 和 SLO 测量收益。

## 总结

Service Mesh 是通信治理平台，不替代业务安全和 Resilience 设计。收益必须覆盖运行与认知成本。
