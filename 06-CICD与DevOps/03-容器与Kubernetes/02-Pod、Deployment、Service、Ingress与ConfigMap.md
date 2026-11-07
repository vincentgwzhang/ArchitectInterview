# 02. Pod、Deployment、Service、Ingress 与 ConfigMap

## Pod

Kubernetes 最小调度单元，共享 Network Namespace 和 Volume。通常一个主业务容器，Sidecar 仅用于紧密生命周期能力。Pod 是短暂的，不能依赖本地身份/磁盘持久性。

## Deployment

管理 Stateless Pod Replica、Rolling Update 和 ReplicaSet。配置 Strategy、Readiness、Resource、SecurityContext 与 Graceful Termination。

Stateful Workload 可使用 StatefulSet，但数据库是否应自建在 K8s 要结合运营能力。

## Service

为动态 Pod 提供稳定 Virtual IP/DNS。类型 ClusterIP、NodePort、LoadBalancer。Selector 错误会无 Endpoint，需监控。

## Ingress/Gateway

管理 HTTP Routing、TLS 和外部入口。Ingress 是 API，实际行为由 Controller 决定；新架构可评估 Gateway API。业务认证和限流不应只依赖默认入口配置。

## ConfigMap/Secret

ConfigMap 保存非敏感配置；Secret 只是 API 对象，默认 Base64 不是加密。启用 At-rest Encryption、RBAC，并优先外部 Secret Manager。配置变更是否自动 Reload 要明确。

## Label

统一 app、version、owner、environment 等 Label，支持 Selector、Cost 和 Observability。避免不可控高 Cardinality。

## 总结

这些对象把运行、发现、入口和配置分离。正确生命周期、权限和不可变部署比 YAML 数量更重要。
