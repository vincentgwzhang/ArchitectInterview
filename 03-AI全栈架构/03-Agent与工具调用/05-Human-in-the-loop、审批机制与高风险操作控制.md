# 05. Human-in-the-loop、审批机制与高风险操作控制

## 为什么需要 Human Oversight

当错误不可逆、影响财务/安全/法律或模型置信不足时，不能让 LLM 自主执行。Human-in-the-loop 不是“所有请求都人工看”，而是按风险设置检查点。

## 风险分级

- Low：生成草稿、只读检索，可自动完成。
- Medium：对外发送内容、低金额变更，可抽样或条件审批。
- High：转账、删除数据、权限提升、法律/医疗决定，必须显式审批或禁止自动化。

## 审批对象

审批 UI 应展示：

- 原始用户目标。
- Agent 计划与拟执行动作。
- 精确参数和目标 Resource。
- 数据来源与 Citation。
- 预期影响和可逆性。
- 风险提示。
- Approve、Edit、Reject。

不要让用户只看到模糊的“Agent 要执行操作，是否同意？”

## 安全执行

审批必须绑定具体 Action Hash、参数和有效期。审批后参数变化需要重新批准，防止 TOCTOU。

高风险动作使用：

- Least-privilege Credential。
- Short-lived Authorization Grant。
- Idempotency Key。
- Dry Run/Preview。
- Two-person Approval（需要时）。
- Audit Trail。
- Reversible/Soft Delete。

## Confidence

模型自报 Confidence 不可靠。Escalation 应结合规则、缺少证据、异常 Tool Result、任务类型和 Eval 校准信号。

## 人工反馈

记录 Edit/Reject 原因，用于改进 Dataset 和 Policy，但必须处理隐私、Label Quality 和 Selection Bias。

## 运营

定义 Approval SLA、Queue Ownership、超时、代理审批、撤回和 Emergency Procedure。审批积压本身是系统可用性风险。

## 总结

Human-in-the-loop 是风险控制架构。精确展示动作、绑定审批参数、最小权限执行和完整审计，才能让人类监督真正有效。
