# 04. 服务目录、API 治理与依赖关系管理

## Service Catalog

每个服务记录：Owner、Repository、Runtime、API/Event、SLO、Dashboard、Runbook、Dependencies、Data Classification、On-call 和 Lifecycle。

目录应从 CI/CD、Kubernetes、OpenAPI 和 Telemetry 自动同步，避免手工 Wiki 过期。

## API Governance

- Naming/Error/Pagination Standard。
- OpenAPI/AsyncAPI Registry。
- Compatibility Check。
- Auth/Security Baseline。
- Deprecation/Sunset。
- Consumer/Usage Visibility。

通过 Template、Lint 和 Gateway 自动化 Guardrail，而不是所有 API 手工委员会审批。

## Dependency Map

静态声明 + Runtime Trace 建图。识别 Critical Path、Single Point、循环依赖和高 Fan-in Service。注意异步 Topic 也形成依赖。

## Ownership

Tier-1 Service 必须有 On-call、SLO、Capacity 和 DR。无 Owner 服务是风险，应 Transfer、Retire 或明确平台托管。

## 生命周期

Proposed → Active → Deprecated → Retired。弃用前确认 Consumer，并监控流量归零。删除包括 DNS、Secret、Data、Dashboard 和 Cost Resource。

## 总结

服务目录提供可发现的 Ownership 和运营元数据；API/依赖治理让自治团队在共同 Guardrail 下安全演进。
