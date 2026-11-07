# 04. OpenAPI、Contract First 与消费者驱动契约测试

## Contract First

先定义 API Contract，再实现 Server/Client，有利于并行开发、Review 与治理。OpenAPI 描述 Path、Schema、Security、Error 和 Example，但业务语义仍需文字说明。

## OpenAPI 治理

- Schema 进入 Version Control。
- Lint 命名、分页、错误与安全规则。
- Compatibility Diff 阻止 Breaking Change。
- 生成 Client/Server Stub，但不要把生成代码作为 Domain Model。
- 发布到 API Catalog。

## Code First

从 Spring Annotation 生成 Spec 上手快，但容易让实现细节主导 Contract。无论哪种方式，Spec 必须成为可验证 Source of Truth，防止文档漂移。

## Consumer-driven Contract

Consumer 发布它依赖的请求/响应期望，Provider 在 CI 中验证。它能发现 Provider 改动是否破坏真实 Consumer。

CDC 不替代 Provider Functional Test、Schema Test 或 E2E。错误 Contract 也会一起通过，因此需要明确 Ownership 和 Broker 治理。

## Mock 与 Sandbox

由同一 Spec 生成 Mock，帮助并行开发。但 Mock 不能证明生产行为、Latency 和 Authorization，仍需 Integration Test。

## Breaking Change

不仅是删除字段：收紧 Validation、改变 Enum、Status Code、Nullability、默认排序和 Error Code 都可能破坏 Consumer。

## 总结

OpenAPI 提供机器可读契约，Contract First 建立协作边界，CDC 验证真实依赖。三者结合支持独立交付和安全演进。
