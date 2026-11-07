# 05. 如何设计兼容旧客户端的 API 演进策略

## 基线

移动端和外部 Client 可能多年不升级，因此 Server 必须知道真实 Version Distribution 和 Contract 使用情况。

## Compatible Change

- 新增 Optional Field/Endpoint。
- 保持旧字段语义、单位、Nullability。
- Consumer 忽略未知字段/Enum。
- 新行为通过 Capability/Feature Negotiation。
- Server 提供 Default。

## Breaking Change

必须发布新 Major Version 或新 Endpoint。旧新版本可共存，共享 Domain Service，避免复制整套业务实现。

## Adapter

在 API Boundary 将 V1/V2 DTO 转换为内部 Command/Model。Internal Model 不被旧 Contract 永久绑死。

## 数据迁移

使用 Expand-and-Contract，先让 Server 支持新旧格式，再更新 Client，观察 Adoption，最后 Sunset。不要先删除字段再等待 Client 更新。

## Deprecation

发布 Timeline、Migration Guide、SDK、Header Warning 和 Usage Dashboard。联系活跃 Consumer；到期前评估业务与安全风险。

## Test

OpenAPI Compatibility、Consumer Contract、旧 Client E2E、Replay Production Request。Canary 观察 Version-specific Error。

## 面试亮点

API 版本只是最后手段。重点是兼容演进、Boundary Adapter、Consumer Visibility 和有证据的弃用流程。
