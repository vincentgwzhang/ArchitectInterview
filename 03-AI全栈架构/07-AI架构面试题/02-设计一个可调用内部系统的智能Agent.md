# 02. 设计一个可调用内部系统的智能 Agent

## 范围

明确 Agent 能读什么、写什么、哪些动作不可逆、是否代表用户身份、审批要求和最大成本。先选择一个有限 Domain，如 IT Support，而不是“能操作所有企业系统”。

## 架构

```text
React Chat
 -> Agent Gateway (Auth, Policy, Budget)
 -> Planner/State Machine
 -> Tool Registry
 -> Tool Runtime (Schema, Authorization, Idempotency)
 -> Internal APIs
 -> Audit/Trace/Approval
```

## Tool

每个 Tool 小而明确，如 `get_ticket`、`prepare_access_request`、`submit_access_request`。禁止任意 SQL/HTTP/Shell。参数 Schema 严格，Runtime 使用用户身份做 Resource Authorization。

## Workflow

只读步骤可自动执行；写操作先生成 Preview，进入 `AWAITING_APPROVAL`。Approval 绑定参数 Hash 和 Expiry。高风险权限变更使用 Two-person Approval。

## State

持久化 Goal、Plan、Tool Call/Result、Approval、Budget 和 Checkpoint。设置 Max Step、Token、Time，检测重复 Loop，支持 Cancel/Resume。

## Security

防 Indirect Prompt Injection；Tool Result 视为不可信；最小权限 Credential；Tenant Isolation；Secret/PII Redaction；完整 Audit 与 Kill Switch。

## Failure

Tool Timeout 不代表未执行。通过 Idempotency Key 和 Status Query 确认。补偿不可逆操作时升级人工，不允许模型自行猜测恢复动作。

## 评估

Task Success、Tool Selection/Argument Accuracy、Unauthorized Attempt Block、Human Edit、Step/Cost、Recovery 和 Safety Attack Success。

## 面试亮点

强调 Agent 不是 Security Principal，Application Policy 才决定权限；优先 Workflow/State Machine，逐步增加自主性。
