# 04. Helm、Kustomize、Operator 与集群配置管理

## Helm

Chart 模板化 Kubernetes Manifest，支持 Package、Dependency、Values 和 Release。适合可复用 Application Package，但复杂模板逻辑难读，渲染后必须验证。

## Kustomize

以 Base + Overlay Patch 组合原生 YAML，不使用模板语言。适合环境差异较小、希望保持资源结构清晰的配置。

## Operator

通过 Custom Resource + Controller 持续 Reconcile 期望状态，适合有复杂生命周期、Backup、Failover、Upgrade 的有状态系统。编写 Operator 相当于开发分布式 Control Loop，成本不低。

## 选择

- 应用打包与分发：Helm。
- 组织已有 YAML 的环境 Overlay：Kustomize。
- 需要持续领域运维自动化：Operator。

可组合，但避免 Helm 中再套复杂 Kustomize 形成难以追踪的生成链。

## 配置治理

- Chart/Module Version。
- JSON Schema 验证 Values。
- Render + Policy Test。
- 环境差异最小化。
- Secret 不放明文 Values。
- GitOps 只部署生成后的明确期望状态。

## 总结

选择最简单能表达生命周期的工具。模板复用、Overlay 和 Controller 各解决不同问题，不应互相替代。
