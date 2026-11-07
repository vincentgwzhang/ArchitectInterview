# 05. Golden Signals、SLO Dashboard 与故障复盘

## Golden Signals

- Latency。
- Traffic。
- Errors。
- Saturation。

从用户 Journey 到 Service/Dependency 分层展示。Average 会隐藏 Tail，应关注 P95/P99 和 Error 类型。

## SLO Dashboard

包含 SLI、目标、Error Budget Remaining、Burn Rate、Release/Event Overlay 和分段维度。SLO 应基于用户成功，而不是 Pod Uptime。

## Alert

Multi-window Burn Rate 捕获快速严重与慢性消耗。Page 对应即时用户影响；Ticket 处理趋势和容量。

## Incident

统一指挥、沟通、Operations 和 Scribe 角色。优先 Mitigate，再 Root Cause。保留 Timeline、Decision 和 Evidence。

## Postmortem

记录 Impact、Timeline、Contributing Factor、What Went Well、Where Lucky 和 Action。Blameless 不等于无 Accountability。

Action 应系统性、有 Owner/Date/验证：自动 Guardrail、隔离、测试、恢复能力优于“以后小心”。

## 学习闭环

同类事故与 Near Miss 聚合分析，更新 SLO、Runbook、Chaos Test、Architecture Standard 和 Roadmap。跟踪 Action Completion 与复发率。

## 总结

Golden Signal 发现影响，SLO 量化可靠性，Postmortem 将事故转化为系统改进。三者形成运行反馈闭环。
