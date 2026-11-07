# 02. 线程、线程池、锁、CAS 与并发容器

## Java Memory Model

并发正确性包含 Atomicity、Visibility 和 Ordering。`synchronized`、`volatile`、Lock 与并发容器建立 Happens-before 关系。`volatile` 保证可见性和部分顺序，不保证 `count++` 原子性。

## 线程池

线程池隔离资源并限制并发：

```text
threads ≈ CPU cores                 # CPU-bound
threads > CPU cores                 # I/O-bound，仍需测量
```

必须配置有界 Queue、Thread Factory、Rejection Policy、Metric 和 Shutdown。无界 Queue 会把过载变成高延迟与 OOM；线程数过大造成 Context Switch，并可能压垮数据库连接池。

## Lock 与 CAS

- `synchronized` 简单且由 JVM 优化。
- `ReentrantLock` 支持 Try Lock、Interrupt 和多个 Condition。
- CAS 通过比较并交换实现 Lock-free Primitive，但要注意 ABA、自旋成本和复杂性。
- `AtomicInteger` 适合简单原子更新，`LongAdder` 在高竞争统计场景吞吐更好但读取不是单一原子快照。

## 并发容器

`ConcurrentHashMap`、BlockingQueue、CopyOnWriteArrayList 等提供不同一致性和性能。`CopyOnWriteArrayList` 适合读多写极少；BlockingQueue 适合 Producer/Consumer 与 Backpressure。

## 常见问题

Deadlock、Starvation、Livelock、Race Condition 和 ThreadLocal Leak。统一 Lock 顺序、缩小 Critical Section、避免持锁调用外部系统，并在线程池任务完成后清理 ThreadLocal。

## 面试要点

不要只说“加锁”。说明共享状态、临界区、竞争程度、失败/中断语义和如何用 JFR、Thread Dump 与 Metric 定位问题。

## 总结

并发设计首先减少共享可变状态，其次使用合适 Primitive，并通过有界资源和监控防止过载扩散。
