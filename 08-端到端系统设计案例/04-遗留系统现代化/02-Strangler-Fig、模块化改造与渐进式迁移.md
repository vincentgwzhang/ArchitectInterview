# 02. Strangler Fig、模块化改造与渐进式迁移

## Strangler Fig

Facade/Gateway 拦截入口，新系统逐个接管 Capability，旧系统范围持续缩小。每个 Slice 可独立验证、回滚和产生价值。

## 模块化

先在 Legacy 内建立 Domain Module、Public API 和依赖规则，减少共享表与循环。即使最终不拆 Microservice，也能改善交付。

## 选择 Slice

边界清晰、变更频繁、业务价值高且风险可控。避免只迁无人使用页面，也避免第一步重写最复杂交易核心。

## 兼容层

Anti-corruption Layer 转换 Legacy Model/API，防止旧概念污染新 Domain。Adapter 有 Owner 和删除日期。

## 切流

Shadow、Dual Read Compare、Canary、Feature Flag。对写操作防止双 Side Effect；Source of Truth 迁移要明确时点。

## 完成定义

旧代码、路由、表、Job、权限、监控和基础设施全部删除，才算完成。只增加新系统会永久双倍成本。

## 总结

渐进迁移通过小步降低风险，但必须持续消除旧路径和临时 Bridge。
