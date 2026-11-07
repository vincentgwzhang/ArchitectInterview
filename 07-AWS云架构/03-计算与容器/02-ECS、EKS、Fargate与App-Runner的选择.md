# 02. ECS、EKS、Fargate 与 App Runner 的选择

## ECS

AWS Native Container Orchestrator，控制面简单，与 IAM/ALB/CloudWatch 集成好。适合希望降低 Kubernetes 运维的团队。

## EKS

Managed Kubernetes Control Plane，适合 Kubernetes 生态、复杂调度、多团队平台和可移植需求。团队仍负责 Node、Addon、Upgrade、Policy、Network 和大量集群运营。

## Fargate

Serverless Container Compute，可用于 ECS/EKS，按 Task/Pod 资源付费，免管理 Node。适合突发、隔离和中等规模；大规模稳定负载可能比 EC2 贵，Daemon/特殊硬件等能力有限。

## App Runner

从 Source/Image 快速部署 Web Service，自动 Build/Scale/TLS，控制少、上手快。适合简单 HTTP 应用，不适合复杂网络/平台需求。

## 选择维度

Workload、规模、Kubernetes 需求、团队技能、Control、启动时间、成本、GPU/Daemon、Multi-tenancy 和生态。

## 决策

默认从最少运营复杂度开始：简单 Web 用 App Runner，AWS Container 用 ECS，已有 Kubernetes Platform/需求用 EKS；Fargate 是计算模式而非编排替代。

## 总结

选择应优化 TCO 和认知负担，不是技术声望。Managed Control Plane 也不等于零运维。
