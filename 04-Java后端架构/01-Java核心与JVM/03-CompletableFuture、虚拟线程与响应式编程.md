# 03. CompletableFuture、虚拟线程与响应式编程

## CompletableFuture

用于组合异步步骤：`thenApply` 转换结果，`thenCompose` 串联异步调用，`allOf` 并行等待。不要无意使用共享 `ForkJoinPool.commonPool` 执行阻塞 I/O，应提供隔离的 Executor。

```java
var product = productClient.get(id);
var stock = inventoryClient.get(id);
return product.thenCombine(stock, ProductView::new)
    .orTimeout(500, MILLISECONDS);
```

异常必须在正确 Layer 处理，并传播 Cancellation 和 Deadline。并行 Fan-out 会增加下游压力，不能只优化单请求 Latency。

## Virtual Thread

Virtual Thread 让大量 I/O-bound 任务使用同步代码风格，降低 Platform Thread 占用。它不会增加 CPU，也不会解决数据库连接、Rate Limit 或外部系统容量。

避免在 Virtual Thread 中长时间执行会 Pin Carrier 的操作，并用 Semaphore/Connection Pool 约束真正稀缺资源。

## Reactive

Reactive Stream 提供 Non-blocking Pipeline 和 Backpressure，适合大量并发 I/O、Streaming 与 Event Flow。代价是 Debug、Context Propagation 和团队学习成本。调用链中混入阻塞代码会破坏优势。

## 选择

- 普通并发与少量组合：同步代码或 CompletableFuture。
- 大量阻塞式 I/O，团队偏同步模型：Virtual Thread。
- 端到端 Non-blocking、Streaming、Backpressure：Reactive。
- CPU-bound：受控 Parallelism，不因换模型自动变快。

## 面试要点

比较 Programming Model、资源限制、Backpressure、Observability 和生态兼容性，而不是宣称某一种方案全面替代其他方案。

## 总结

三者都是并发组织方式。真正容量由 CPU、连接、下游 Quota 和数据层决定，必须配合 Timeout、隔离和测量。
