# 04. Tagging、配额、资源治理与责任分配

## Tagging

标准 Tag 可包含 `owner`、`application`、`environment`、`cost-center`、`data-classification`、`managed-by`。Tag 支持成本分摊、自动化、搜索和部分 IAM，但不是所有服务/操作都支持相同语义。

使用 Tag Policy、IaC Default Tag 和 Config 检查。不要把敏感数据放入 Tag，因为它可能广泛可见。

## Service Quota

Quota 是隐藏的容量边界，如 VPC、ENI、Elastic IP、API Rate、Lambda Concurrency。上线前做 Quota Review，按增长提前申请，提高限额不等于下游有容量。

## 资源治理

- 所有生产资源由 IaC 管理。
- 禁止未批准 Region/Service。
- 自动清理过期 Preview/Sandbox。
- Resource Catalog 与 Owner。
- Drift/Orphan Detection。
- Lifecycle/Backup Policy。

## Responsibility

Cloud Platform 拥有 Landing Zone 和 Guardrail；Workload Team 拥有应用、SLO、成本和数据；Security 定义控制并监控。Shared Responsibility 必须细化到组织内部 RACI。

## 总结

Tag 提供元数据，Quota 定义容量边界，自动治理维持一致性，明确 Owner 才能把发现的问题转化为行动。
