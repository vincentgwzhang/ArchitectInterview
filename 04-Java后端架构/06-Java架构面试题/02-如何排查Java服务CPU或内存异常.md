# 02. 如何排查 Java 服务 CPU 或内存异常

## 先确认影响

查看 SLO、Latency、Error、Traffic、Version 和 Region。区分应用 CPU、高 GC CPU、Container Throttling、Host Noise 与依赖导致的 Busy Loop。

## CPU 流程

1. 看 CPU、Load、Thread Count、GC、Request Rate。
2. 获取 JFR/Async-profiler Flame Graph。
3. Thread Dump 对照高 CPU Thread。
4. 判断热点是业务计算、Serialization、Regex、Lock Spin、GC 或 Logging。
5. Mitigate：Rollback、限流、扩容；再修 Root Cause。

## Memory 分类

- Heap Leak：Full GC 后 Baseline 持续增长。
- Allocation Pressure：频繁 Young GC，回收后正常。
- Native/Direct/Metaspace。
- Thread Stack。
- Container OOMKill。

## Memory 流程

1. 对比 Heap Used、RSS、GC、Direct Buffer、Thread。
2. 安全生成 Heap Dump/JFR。
3. Dominator Tree 与 Retainer Path 找最大 Retained Object。
4. 检查 Cache、Listener、ThreadLocal、ClassLoader、Queue。
5. Native Memory Tracking 处理非 Heap。

## 注意

Heap Dump 含敏感数据，需受控存储。生产不要随意执行长时间 Stop-the-world 工具。先保证服务恢复，再做深入分析。

## 面试亮点

展示分类诊断和证据，不以“重启”和“加大内存”作为最终解决方案。
