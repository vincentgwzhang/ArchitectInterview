# 05. GC 算法、内存泄漏、性能分析与 JVM 调优

## GC 目标

GC 在 Throughput、Pause、Footprint 和 CPU 之间权衡。没有脱离 Workload 的最佳 Collector。

- G1：通用 Server 默认选择，按 Region 管理并控制 Pause Goal。
- ZGC/Shenandoah：面向大 Heap 和低 Pause，代价与兼容性需实测。
- Serial/Parallel：简单或强调 Throughput 的特定场景。

## 调优顺序

1. 定义 P99 Latency、Throughput 和 Memory SLO。
2. 收集 GC Log、JFR、Allocation Rate、Heap Occupancy。
3. 判断是 Allocation、Leak、Promotion、Pause 还是 CPU 问题。
4. 先修代码、数据结构与 Cache。
5. 再调整 Heap、Collector 和参数。
6. 用生产式 Load Test 验证。

不要复制网络上的大量 JVM Flag。现代 JVM 默认值通常不错，错误参数可能阻止自动优化。

## Leak 与压力

- Leak：Old Gen 基线持续上升，Full GC 后不回落。
- 高 Allocation：回收后基线稳定，但 GC 频繁。
- 内存不足：Live Set 接近 Heap Limit。

使用 Heap Dump 的 Dominator/Retainer Path 找到谁持有对象，而不只看实例数量。

## 容器配置

保留 Native Headroom，设置合理 `-Xms/-Xmx` 或 MaxRAMPercentage，监控 RSS、GC、CPU Throttling。OOM 前自动生成 Heap Dump 时注意磁盘空间和敏感数据。

## 面试要点

展示证据驱动过程：Metric → Hypothesis → Profile → Change → Load Test，而不是“把 Heap 调大”。

## 总结

JVM 调优的核心是满足业务 SLO。先理解 Allocation 与 Live Set，再选择 Collector 和容量，最后通过真实负载验证。
