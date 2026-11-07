# 03. 如何提高慢速 Pipeline 的反馈效率

## 先测量

分解 Queue、Checkout、Dependency、Compile、Test、Image、Scan 和 Deploy，查看 P50/P95、Critical Path、Failure Stage 和 Cache Hit。

## 优化顺序

1. 快检查前置并 Fail Fast。
2. 独立 Stage 并行。
3. Dependency/Build Layer/Remote Cache。
4. Monorepo Affected-only。
5. Test Sharding 与平衡分片。
6. 预构建 Runner Image。
7. 右配 Runner CPU/Memory。
8. 治理 Flaky 和慢测试。

## 测试分层

PR 跑快且高信号的 Unit/Component/Contract；完整 E2E/Performance 在 Merge、Preview 或定时执行。高风险变更可动态增加测试。

## Queue

Autoscale Ephemeral Runner、按 Workload Pool 隔离，避免长性能测试阻塞普通 PR。设置并发上限控制成本。

## Developer Feedback

本地可运行相同命令，错误链接到具体文件/测试，失败 Artifact 易获取。取消同 Branch 旧 Pipeline。

## Guardrail

不通过删除测试换速度。监控 Escaped Defect、Change Failure 和 Pipeline Cost，确保优化没有降低质量。

## 面试亮点

把 Pipeline 当性能系统，用 Critical Path 和数据优化，同时保护风险型测试覆盖。
