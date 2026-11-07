# 05. 微前端、Module Federation、独立部署与团队边界

## 为什么使用 Micro-frontend

Micro-frontend 的主要驱动力是组织与交付自治，而不是页面技术炫技：

- 多团队拥有不同 Business Domain。
- 需要独立 Build、Test 和 Deploy。
- 大型单体前端发布协调成本过高。
- 渐进迁移遗留技术。

如果只有一个小团队，模块化单体前端通常更简单。

## 集成模式

- Build-time Package：简单、类型安全，但发布仍耦合。
- Runtime Module Federation：独立部署，共享 Remote Module；需要处理版本和运行时故障。
- Route-level Composition：不同 Route 由不同应用负责，边界清晰。
- Server/Edge Composition：Server 组合 Fragment，适合某些内容型页面。
- iframe：隔离强，但 UX、通信和可访问性成本高。

## Module Federation

Host 在 Runtime 加载 Remote。必须治理：

- Shared Dependency 和 React Singleton。
- Version Compatibility。
- Remote Unavailable Fallback。
- Cache Busting 和 Rollback。
- Contract Test。
- Security 与 Supply-chain Trust。

独立部署不等于可以任意破坏 Runtime Contract。

## 团队与 Domain Boundary

按 Business Capability 划分，例如 Catalog、Checkout、Account。不要按 Header Team、Button Team 这类横向 UI 层拆分，否则一次 Feature 仍需多个团队协作。

Shell 通常拥有：

- Routing。
- Authentication Context。
- Global Layout。
- Design Token。
- Error Boundary 与 Telemetry。

Domain Team 拥有其 Feature、API Contract、Test 和 Production Support。

## 共享能力

通过版本化 Design System 和 Platform API 共享。避免巨大 `shared` Package 强制所有应用同步升级。

跨应用状态应最小化，优先 URL、Backend 或明确 Event Contract。直接共享 Global Store 会重新产生 Distributed Monolith。

## 运营要求

- 每个 Remote 独立 Dashboard 和 Owner。
- Shell 对 Remote 设置 Error/Timeout Boundary。
- 支持 Canary 与快速 Rollback。
- E2E 覆盖关键跨域 Journey。
- 监控不同版本组合。

## 何时不要使用

- 单团队、低复杂度应用。
- Domain Boundary 不清。
- Platform 和 CI/CD 尚不成熟。
- 所有 Feature 必须同步发布。
- 性能预算极严且无法接受 Runtime Overhead。

## 总结

Micro-frontend 是组织架构决策。只有当独立 Ownership 和 Deployment 的收益大于 Runtime、UX、Governance 与 Platform 成本时才值得采用。
