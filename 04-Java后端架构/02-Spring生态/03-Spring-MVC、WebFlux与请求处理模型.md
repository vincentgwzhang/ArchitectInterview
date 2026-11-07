# 03. Spring MVC、WebFlux 与请求处理模型

## Spring MVC

典型 Servlet Thread-per-request 模型，生态成熟，适合大多数 CRUD 和传统阻塞式数据库/Client。配合 Virtual Thread 可提高 I/O 并发，但下游资源仍需限制。

## WebFlux

基于 Reactive Stream 与 Event Loop，支持 Non-blocking I/O 和 Backpressure。适合 Streaming、Gateway 和端到端 Reactive Workload。若在 Event Loop 上调用 JDBC 或阻塞 SDK，会阻塞少量关键线程并导致大面积延迟。

## 选择标准

- Dependency 是否 Non-blocking？
- 是否需要大量长连接/Streaming？
- 团队是否熟悉 Reactive Debug 与 Context？
- 现有 Library 是否支持？
- 性能瓶颈是否真的在线程？

不要为了高并发默认使用 WebFlux。MVC + 合理线程/连接池通常更简单。

## 请求生命周期

设计关注：

- Request Size Limit。
- Authentication/Authorization Filter。
- Validation。
- Deadline/Timeout。
- Correlation/Trace Context。
- Exception Mapping。
- Response Streaming 与 Client Disconnect。

## Backpressure

Reactive Backpressure 只在支持的调用链中传播。外部 HTTP、Kafka、数据库和 Client Buffer 都可能形成边界，必须监控 Queue 与 Memory。

## Error 与 Context

统一 Problem Details，保留 Trace ID。Reactive 中 ThreadLocal 不可靠，使用 Reactor Context；Virtual Thread 也需确认 MDC/Tracing Library 的兼容性。

## 总结

MVC 强调简单成熟，WebFlux 强调端到端 Non-blocking 与 Streaming。架构选择应基于依赖、Workload、团队和测量，而不是框架标签。
