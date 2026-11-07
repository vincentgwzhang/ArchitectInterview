# 01. Prompt Injection、越权工具调用与数据泄露

## Prompt Injection

攻击者通过 User Input、网页、文档、邮件或 Tool Result 注入指令，诱导模型忽略原规则、泄露数据或调用危险 Tool。

- Direct Injection：用户直接输入恶意指令。
- Indirect Injection：恶意内容隐藏在被检索文档、网页或邮件中。

LLM 无法可靠地仅靠 Prompt 区分“可信指令”和“不可信数据”，必须在 Application 层建立边界。

## 防御原则

1. 所有外部内容视为不可信 Data。
2. Tool 使用最小权限与 Allowlist。
3. 执行前做 Schema、Authorization 和 Business Validation。
4. 高风险动作 Human Approval。
5. Retrieval 前执行 Tenant/ACL Filter。
6. 输出经过 DLP、Schema 和 Policy 检查。
7. 限制 Step、Token、Time 和 Tool Budget。

“在 System Prompt 中写不要听从文档指令”只能作为一层软防护。

## 越权 Tool Call

模型可能根据恶意内容调用 `delete_user` 或读取其他 Tenant。Tool Runtime 必须使用当前用户身份重新授权，不能信任模型生成的 `userId` 或 `tenantId`。

高风险 Tool 应拆成 Preview 与 Commit：

```text
prepare_transfer -> show exact amount/account -> approve -> execute_transfer
```

审批绑定参数 Hash，修改后重新审批。

## 数据泄露路径

- Prompt/Response Log。
- Cross-tenant Cache。
- Vector Retrieval ACL 错误。
- Tool 返回过多字段。
- Model Provider Data Policy。
- Citation 链接权限不一致。
- 长期 Memory。
- Error/Trace 和 Analytics。

## 测试

建立攻击 Corpus：角色覆盖、指令混淆、编码、文档隐藏指令、Tool 参数注入、跨租户 Query 和 Exfiltration。评估 Attack Success Rate，而不只是正常任务准确率。

## Incident

准备关闭 Tool、撤销 Credential、隔离 Conversation、保存 Evidence、通知 Data Owner 和重新评估受影响记录的 Runbook。

## 总结

Prompt Injection 无法靠单一 Prompt 彻底解决。真正的防线是权限、隔离、最小 Tool、审批、输出控制和持续红队测试。
