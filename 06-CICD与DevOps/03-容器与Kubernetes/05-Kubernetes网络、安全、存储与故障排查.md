# 05. Kubernetes 网络、安全、存储与故障排查

## 网络

Pod 获得 IP，Service 提供稳定发现，CNI 实现网络。NetworkPolicy 默认允许与否取决于集群，建议 Default Deny 后显式放行必要路径。

排查顺序：DNS → Service/Endpoint → NetworkPolicy → Pod Port/Readiness → Ingress/LB → 外部依赖。

## 安全

- Namespace/RBAC 最小权限。
- Workload Identity，避免静态 Cloud Key。
- Pod Security Standard。
- Non-root、Drop Capability、Read-only FS、Seccomp。
- Admission Policy、签名镜像、Secret Manager。
- 控制 API Server 和 Node Access。

## 存储

PV/PVC/StorageClass 解耦请求与实现。理解 Access Mode、Zone Binding、Snapshot、Backup 和 Reclaim Policy。StatefulSet 稳定身份不等于数据自动备份。

## 排查

```text
kubectl describe -> Event
logs/current+previous -> application
get endpoints -> routing
resource/metric -> saturation
node/pod condition -> scheduling/runtime
```

常见：ImagePull、CrashLoop、OOMKilled、Pending、Probe Failure、DNS、Volume Attach。

## 运营

集群/Node Upgrade、CNI/CSI 版本、Certificate、Quota、Audit 和 DR 都需 Owner。不要把 Kubernetes 当成自动消除运维。

## 总结

Kubernetes 故障跨应用、调度、网络、安全和存储。结构化排查与最小权限平台能力比记命令更重要。
