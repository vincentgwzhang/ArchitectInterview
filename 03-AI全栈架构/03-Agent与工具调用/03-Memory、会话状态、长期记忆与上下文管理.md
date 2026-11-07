# 03. Memory、会话状态、长期记忆与上下文管理

## Memory 类型

- Working Memory：当前请求的 Prompt/Tool Context。
- Conversation State：当前会话历史和摘要。
- User Profile：明确保存的偏好与设置。
- Episodic Memory：过去交互中的事件。
- Semantic Memory：提取后的长期事实或知识。

“把所有聊天记录永久塞进 Prompt”不是 Memory Architecture。

## 会话状态

保存 Message、Tool Call、Result、Model/Prompt Version 和 Token Usage。使用稳定 Conversation ID，并处理并发消息、重试和顺序。

长会话需要：

- Recent Window。
- 摘要旧消息。
- 按相关性检索历史。
- 保留关键 User Constraint。
- Token Budget。

摘要是有损的，应保存原始记录用于审计或重新生成，但按 Retention Policy 管理。

## 长期记忆

只有明确价值且用户授权的信息才保存。每条 Memory 应包含：

- 内容与类型。
- 来源。
- Confidence。
- Created/Updated/Expiry。
- User/Tenant Scope。
- 删除与纠正方式。

模型推断不应直接当成事实写入 Profile。例如“用户可能喜欢旅行”必须和用户明确偏好区分。

## Retrieval

可结合时间、语义、重要性和显式 Pin。避免不相关旧信息污染当前回答，并防止一个用户的 Memory 泄露给另一个用户。

## 隐私

- 明确告知保存什么。
- 提供查看、纠正、删除。
- 最小化 PII。
- 加密和严格 ACL。
- Retention/Expiry。
- 不把 Secret 作为 Memory。

## Context Budget

为 System Instruction、User Query、Recent History、Retrieved Memory、Knowledge 和 Tool Result 分配预算。超限时按业务优先级压缩，不要简单从最前面截断导致规则丢失。

## 总结

Memory 是有 Ownership、Retention 和质量控制的数据系统。区分会话、偏好、事实与推断，才能兼顾个性化、成本、正确性与隐私。
