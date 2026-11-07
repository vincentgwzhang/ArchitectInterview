# 05. Pipeline as Code、并行执行、缓存与反馈速度

## Pipeline as Code

Pipeline 进入 Version Control，经过 Review、Test 和版本化。共享 Template/Component 提供 Golden Path，但业务仓库保留清晰可见的调用配置。

## DAG 与并行

建模真实依赖：Lint、Unit、SCA 可并行；Package 等必要检查后执行。动态矩阵按 OS/Version 测试，限制无价值组合。

## Cache

缓存 Dependency、Compiler Output、Docker Layer 和 Monorepo Task。Cache Key 包含 Lockfile、Toolchain、配置和 Source Hash。Cache 不是 Artifact Source of Truth，失败时必须能干净重建。

防止不可信 PR 写入共享高权限 Cache，避免 Cache Poisoning。

## 速度优化

- Affected-only Test。
- Test Sharding。
- Remote Build Cache。
- 预构建 Runner Image。
- Fail Fast。
- 合理资源规格。
- Flaky/慢测试治理。

先 Profile Pipeline Critical Path，不要只增加 Runner。

## 可观测性

统计 Queue Time、Stage Duration、Cache Hit、Failure Category、Flake、Cost 和 Developer Wait Time。P95 比平均值更能反映体验。

## Template 版本

共享 Pipeline Template 用版本固定和自动升级 PR，避免一次中央变更破坏所有仓库。

## 总结

Pipeline 是生产系统。代码化、DAG、可信缓存、性能指标和版本化模板共同提供快速可靠反馈。
