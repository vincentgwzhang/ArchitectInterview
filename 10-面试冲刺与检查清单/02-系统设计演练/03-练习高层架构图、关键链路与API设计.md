# 03. 练习高层架构图、关键链路与 API 设计

## 画图顺序

1. Actor/Client 与 System Boundary。
2. 入口、核心 Service、Data Store。
3. 同步/异步箭头与 Protocol。
4. Critical User Journey Sequence。
5. Cache/Queue/External Dependency。
6. Deployment/Failure/Security。

## API

为 2–3 个核心 Use Case 定义 Method、Path、关键 Request/Response、Error、Idempotency 和 Pagination，不必列全系统。

## 关键链路

指出 Transaction Boundary、Source of Truth、Latency Budget、Timeout/Retry 和 Partial Failure。正常流程讲完后主动讲异常。

## 图的质量

每个 Box 有职责，每个箭头有方向和语义；逻辑图与 AWS 部署图分开。边画边解释 Decision。

## 练习

用白板/在线工具在 15 分钟内完成一张主图和一条 Sequence，录屏检查是否沉默、跳跃或过度细节。

## 总结

系统图是沟通工具。主图应让面试官看懂边界和数据流，深度通过关键链路展开。
