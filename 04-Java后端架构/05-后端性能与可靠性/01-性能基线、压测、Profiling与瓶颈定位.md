# 01. 性能基线、压测、Profiling 与瓶颈定位

## Baseline

定义 Workload、数据规模、环境和目标：RPS、P50/P95/P99、Error、CPU、Memory、DB/Dependency。没有可重复 Baseline，优化结果无法比较。

## 测试类型

- Load：目标 Peak。
- Stress：寻找 Break Point。
- Spike：突发。
- Soak：长期 Leak/退化。
- Failover：组件故障后的容量。

测试包含真实读写比例、数据倾斜、认证、Cache Warm/Cold 和 Dependency Latency。

## Profiling

- JFR/Async-profiler：CPU、Allocation、Lock、I/O。
- GC Log/Heap Dump：Memory。
- Trace：跨 Service Critical Path。
- Database Plan/Slow Query：数据层。
- Thread Dump：Block、Deadlock、Pool Exhaustion。

## 方法

```text
Metric -> 定位 Layer -> Profile -> Hypothesis
-> 单一 Change -> 重测 -> 检查副作用
```

关注 Saturation Point；接近容量时 Queue 和 Tail Latency 非线性增长。

## 常见错误

- 在 Laptop 用空数据库得出生产容量。
- 只看 Average。
- 压测工具自身成为 Bottleneck。
- 调 JVM 参数而不看 Profile。
- 优化一个 Instance，却忽略 Database/External API。

## 总结

性能工程是证据驱动循环。生产式负载、分层 Metric 和精确 Profile 比经验猜测更可靠。
