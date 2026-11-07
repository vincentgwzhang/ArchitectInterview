# 02. Java/Python AI Gateway、模型适配层与 API 设计

## Gateway 职责

AI Gateway 统一：

- Provider/Model Routing。
- Auth、Tenant、Quota、Rate Limit。
- Prompt/Policy Version。
- Streaming Protocol。
- Retry、Timeout、Fallback。
- Token/Cost/Trace。
- Safety Filter 与 Data Redaction。

不要让每个业务 Service 直接集成多个 Model SDK，形成重复逻辑和难以治理的数据流。

## Java 与 Python

- Java：企业 API、Spring Security、Transaction、强类型、现有 Microservice 生态。
- Python：AI Library、Data Pipeline、Experiment 与 Model Serving 生态。

常见组合：Java 负责 Product API 和业务权限，Python 负责 Retrieval/Model Workflow，通过 versioned REST/gRPC/Event Contract 协作。也可按团队能力使用单一语言，避免无意义 Polyglot。

## Adapter

定义内部统一 Contract，但不要过度抽象到所有 Provider 的最低共同能力：

```text
generate(request)
stream(request)
embed(batch)
invokeTools(request)
```

保留 Provider-specific Capability 的受控扩展。不同 Model 对 Tool、Reasoning、Multimodal 和 Structured Output 支持不同。

## API

请求包含 Conversation/Request ID、Use Case、Input、Allowed Tools、Response Schema 和 Deadline。服务端决定具体 Model，Client 不应随意指定昂贵或不合规 Model。

Streaming 使用清晰 Event Envelope 与 Version。错误区分 Validation、Policy、Rate Limit、Provider、Timeout 和 Unknown Outcome。

## Reliability

- Per-provider Timeout/Circuit Breaker。
- 有界 Retry，只对安全错误。
- Bulkhead 与 Tenant Quota。
- Fallback 要通过质量评测。
- Streaming Backpressure 与 Disconnect Cancellation。

## Security

集中执行 PII Redaction、Data Residency、Allowed Model、Tool Permission 和 Audit。Prompt/Response Log 默认不保存敏感全文。

## 总结

AI Gateway 把快速变化的 Model Provider 与业务系统隔离，同时集中治理安全、成本和可靠性。抽象应统一共同策略，但保留能力差异。
