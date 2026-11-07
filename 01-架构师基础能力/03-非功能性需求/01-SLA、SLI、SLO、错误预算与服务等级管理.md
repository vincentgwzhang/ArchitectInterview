# 01. SLA、SLI、SLO、错误预算与服务等级管理

## 学习目标

1. 区分 SLA、SLI、SLO 与 Error Budget。
2. 为关键 User Journey 选择有效指标。
3. 正确计算 Availability 与 Burn Rate。
4. 用 Error Budget 平衡 Reliability 和 Feature Delivery。

## 一、四个核心概念

### 1.1 SLI：Service Level Indicator

SLI 是对服务行为的定量测量，例如：

- 成功请求比例。
- P95/P99 Latency。
- 正确处理的 Event 比例。
- 新鲜数据比例。
- Durability。

常见形式：

```text
SLI = Good events / Valid events
```

例如 1,000,000 个有效请求中有 999,500 个在 300 ms 内成功：

```text
Latency-and-success SLI = 999,500 / 1,000,000 = 99.95%
```

### 1.2 SLO：Service Level Objective

SLO 是 SLI 在指定 Window 内的目标：

> 过去 28 天中，99.9% 的有效 Checkout 请求在 500 ms 内成功完成。

SLO 必须包含：

- 测量对象。
- Good Event 定义。
- 目标百分比。
- 时间窗口。
- 排除规则。

### 1.3 SLA：Service Level Agreement

SLA 是与客户或业务方之间的正式承诺，通常包含违约后果，如 Service Credit。

SLA 一般应比内部 SLO 宽松，为监控误差、运营响应和意外波动保留空间。

### 1.4 Error Budget

Error Budget 是 SLO 允许的不可靠程度：

```text
Error Budget = 1 - SLO
```

99.9% Availability 在 30 天内允许约：

```text
30 × 24 × 60 × 0.1% = 43.2 minutes
```

常见近似：

| SLO | 30 天允许不可用时间 |
| --- | ---: |
| 99% | 7 小时 12 分钟 |
| 99.9% | 43 分 12 秒 |
| 99.95% | 21 分 36 秒 |
| 99.99% | 4 分 19 秒 |

## 二、为什么不能只看 Uptime

服务进程在线不表示用户能够完成任务。可能出现：

- API 返回 200，但内容错误。
- Checkout 可访问，但支付一直超时。
- Consumer 在线，但 Event Lag 已达到两小时。
- 页面加载成功，但关键按钮不可用。

因此应围绕 User Journey 定义 SLI，而不是只监控 Host 或 Pod。

例如订单系统可定义：

- 创建订单成功率。
- 创建订单 P99 Latency。
- Payment Callback 5 分钟内完成比例。
- Order Event 60 秒内被处理的比例。

## 三、Availability 的正确口径

请求型系统可以定义：

```text
Availability = successful valid requests / total valid requests
```

需要提前定义：

- 4xx 是否计入失败？通常用户输入导致的 4xx 不计入服务错误。
- Dependency Failure 是否计入？如果用户无法完成流程，通常应该计入。
- Planned Maintenance 是否排除？取决于 SLA/SLO 定义。
- Synthetic Traffic 是否计入？通常单独观察。
- Partial Success 如何处理？应根据业务语义定义。

如果使用“不可用分钟数”，一次只影响 5% 用户的故障与全站故障可能被同样计算。基于请求或用户 Journey 的 SLI 通常更精确。

## 四、Latency SLO

Average Latency 会掩盖 Tail Latency。应关注 Percentile：

- P50：典型用户体验。
- P95：大多数用户体验。
- P99/P99.9：Tail 和异常行为。

一个有效目标：

> 28 天窗口内，99% 的 Search 请求在 800 ms 内返回有效结果。

它把成功率与 Latency 结合，避免系统用快速错误响应“满足”性能目标。

## 五、Window 选择

常见 Window：

- Rolling Window：任意时刻查看过去 28/30 天，持续反映当前状态。
- Calendar Window：按自然月计算，适合 SLA 和财务结算。

Rolling Window 更适合工程运营。Window 太短会对偶发波动过敏，太长则使改进反馈缓慢。

## 六、Error Budget Policy

Error Budget 让 Reliability 讨论从情绪变成策略：

- Budget 充足：可以进行 Feature Release、Experiment 和 Planned Change。
- Budget 消耗过快：加强 Review、减少风险发布、优先 Reliability Work。
- Budget 已耗尽：暂停非必要变更，聚焦恢复和系统性改进。

Policy 应由 Product、Engineering 和 SRE 共同制定，不能由单一团队机械执行。

典型政策：

```text
如果 28-day Error Budget 消耗超过 100%，
除 Security Fix 和关键 Business Change 外暂停生产发布，
直到 Burn Rate 回到目标并完成责任明确的修复计划。
```

## 七、Burn Rate

Burn Rate 表示 Error Budget 消耗速度：

```text
Burn Rate = observed error rate / allowed error rate
```

SLO 为 99.9%，允许错误率 0.1%。如果当前错误率为 1%：

```text
Burn Rate = 1% / 0.1% = 10
```

意味着以当前速度，Budget 会比计划快 10 倍耗尽。

Multi-window Alert 可以同时捕获快速严重故障和慢性问题：

- 短 Window + 高 Burn Rate：Page，立即响应。
- 长 Window + 中等 Burn Rate：Ticket，计划修复。

避免对瞬时单点错误产生大量 Alert Noise。

## 八、Dependency SLO

端到端 Availability 受依赖链影响。三个串行依赖各为 99.9%，粗略组合为：

```text
0.999 × 0.999 × 0.999 ≈ 99.7%
```

这说明：

- 用户 Journey SLO 不能高于关键依赖能够支持的水平。
- 可通过 Cache、Fallback、Asynchronous Flow 或 Dependency Redundancy 降低耦合。
- 内部 Service 的目标应根据端到端目标分配，而不是全部统一为 99.99%。

## 九、SLO 设计流程

1. 选择关键 User Journey。
2. 定义用户视角的 Good Event。
3. 确定数据源和测量点。
4. 观察 Baseline，而不是立即设理想值。
5. 与业务讨论失败影响和成本。
6. 设置可实现但有约束力的目标。
7. 建立 Dashboard、Burn Rate Alert 和 Error Budget Policy。
8. 定期 Review 目标是否仍匹配业务。

## 十、常见错误

- 把 SLA、SLO 和 SLI 混用。
- 所有 Service 都设成 99.99%。
- 用 CPU、Pod Count 等内部指标当作用户 SLI。
- 只看 Average Latency。
- SLO 没有时间窗口和排除规则。
- Error Budget 只用于惩罚团队。
- 指标数据不可靠，却用于正式 SLA。

## 十一、面试回答示例

> 对 Checkout，我会定义“有效请求在 800 ms 内成功创建唯一订单”的 SLI，28-day SLO 为 99.95%。客户端 Validation Error 不计为服务失败，但下游支付超时会计入，因为用户无法完成任务。我们使用 Request-based Measurement 和 Multi-window Burn-rate Alert。Error Budget 消耗过快时，暂停高风险发布并优先处理可靠性问题。

## 十二、自测题

1. SLI、SLO 和 SLA 的关系是什么？
2. 99.9% 月度 SLO 大约有多少不可用预算？
3. 为什么 Process Uptime 不是理想的用户 SLI？
4. Burn Rate 为 10 表示什么？
5. 为什么内部 SLO 通常应比外部 SLA 更严格？

## 十三、总结

- SLI 是测量，SLO 是目标，SLA 是带后果的承诺。
- SLO 应围绕关键 User Journey，而不是单纯基础设施状态。
- Error Budget 把可靠性转化为可管理的产品决策。
- Burn Rate 用于识别 Budget 消耗速度并设计有效告警。
- 更高 SLO 意味着更高设计、运维和机会成本，必须由业务价值驱动。
