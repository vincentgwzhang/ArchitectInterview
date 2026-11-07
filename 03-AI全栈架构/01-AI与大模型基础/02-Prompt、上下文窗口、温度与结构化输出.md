# 02. Prompt、上下文窗口、温度与结构化输出

## Prompt 结构

生产 Prompt 应明确分层：

- System/Developer Instruction：角色、边界和安全规则。
- User Input：真实请求，视为不可信数据。
- Retrieved Context：带来源与权限的参考资料。
- Tool Result：结构化外部事实。
- Output Contract：格式、Schema 和失败方式。

不要把不可信文档内容与高优先级 Instruction 混为一体。

## Context Window

Context 包含输入与输出 Token。内容越多不一定质量越高：噪音会稀释证据，增加 Latency、Cost 和“Lost in the Middle”问题。

Context Engineering 包括：

- 选择相关历史消息。
- Summarization。
- Retrieval。
- Tool Result 压缩。
- Token Budget 分配。
- 删除重复或低价值内容。

## Temperature

Temperature 调整 Sampling Distribution。低值更稳定，适合 Extraction、Classification 和 Tool Parameter；较高值增加多样性，适合 Brainstorming。

Temperature 不能让模型变得“更正确”。需要事实准确时，应使用 Retrieval、Tool、Validation 和低随机性。

## 结构化输出

使用 JSON Schema 或 Typed Output，而不是要求模型“输出 JSON 看起来像这样”。

```json
{
  "type": "object",
  "required": ["category", "confidence"],
  "properties": {
    "category": { "enum": ["billing", "technical", "other"] },
    "confidence": { "type": "number", "minimum": 0, "maximum": 1 }
  }
}
```

仍需进行 Runtime Validation、业务规则检查和安全处理。Schema 合法不等于内容正确。

## Prompt 管理

- 模板 Version Control。
- 参数化而非字符串拼接。
- 记录 Model、Prompt Version、Retrieval Version。
- Offline Eval 后再发布。
- 对敏感变量做 Redaction。
- 设置最大输入/输出 Budget。

## 常见错误

- 把整份数据库记录塞入 Prompt。
- 依靠“不要泄密”替代权限控制。
- Prompt 中存在互相冲突 Instruction。
- 解析自由文本驱动高风险操作。
- 修改 Prompt 后没有 Regression Eval。

## 总结

Prompt 是应用 Contract 的一部分。通过清晰层次、受控 Context、合适 Sampling、Schema 和版本评测，才能从 Demo 走向稳定生产。
