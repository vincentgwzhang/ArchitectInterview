# 04. JVM 内存模型、类加载与对象生命周期

## Runtime Memory

- Heap：对象与数组，GC 主要管理区域。
- Thread Stack：Frame、局部变量和调用状态。
- Metaspace：Class Metadata。
- Code Cache：JIT 编译代码。
- Native/Direct Memory：NIO Buffer、JNI 等。

容器 Memory Limit 必须覆盖 Heap 之外的内存。只设置 `-Xmx` 接近 Pod Limit 可能因 Direct Memory、Thread Stack 或 Metaspace 导致 OOMKill。

## 对象生命周期

对象通常在 Eden 分配，存活后进入 Survivor/Old Generation；具体行为取决于 Collector。Escape Analysis 可能消除分配或进行标量替换。不要假设所有 `new` 都必然产生长期 Heap 压力。

只要从 GC Root 可达，对象就不会回收。Memory Leak 在 Java 中通常是“不再需要但仍可达”，如无界 Cache、Listener、ThreadLocal 和 ClassLoader 引用。

## 类加载

阶段包括 Loading、Linking、Initialization。Parent Delegation 防止核心 Class 被任意替换。Application Server、Plugin 和热部署场景可能出现 ClassLoader Leak。

类初始化有同步语义，静态初始化执行昂贵或外部操作会拖慢启动并引发失败。

## Java Memory Model

JMM 定义线程如何观察内存，Heap/Stack 图并不能替代 Happens-before。正确同步保证写入对读取可见。

## 排查

- Heap Dump + Dominator Tree 查 Retained Size。
- Native Memory Tracking 查非 Heap。
- `jcmd`、JFR、GC Log。
- Container RSS 与 JVM Metric 对比。

## 面试要点

能区分 Heap OOM、Metaspace OOM、Direct Buffer OOM 和 Container OOMKill，并说明诊断证据。

## 总结

理解 JVM 全部内存区域、对象可达性和 ClassLoader，才能正确设置容器资源并定位复杂内存问题。
