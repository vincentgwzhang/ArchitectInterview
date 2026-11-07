# 01. React 聊天界面、流式响应、取消与重试

## 状态模型

Message 不应只有 `text`：

```ts
type Message = {
  id: string;
  role: 'user' | 'assistant' | 'tool';
  status: 'pending' | 'streaming' | 'complete' | 'error';
  content: string;
  citations?: Citation[];
  requestId: string;
};
```

显式状态支持 Retry、Cancel、Partial Output 和错误恢复。

## Streaming

常用 SSE 或 Fetch Stream。Client 增量解析 Event，不要假设一个 Network Chunk 等于一个完整 JSON Event。协议应包含：

- `message.started`。
- `content.delta`。
- `tool.started/completed`。
- `citation`。
- `message.completed`。
- `error`。

## UI 更新

每个 Token 都 `setState` 可能频繁 Render，可按 Animation Frame 或小时间窗口 Batch。保留 Scroll Position：只有用户位于底部时自动跟随，否则显示“新内容”提示。

## Cancel

使用 `AbortController` 中止前端 Request，并让 Backend 传播 Cancellation 到 Model/Tool，减少无效成本。取消后保留 Partial Response 还是删除，需要明确 UX。

## Retry

用户 Retry 应使用新 Request ID，并关联 Parent Request。不要自动重试已经产生 Side Effect 的 Tool Call。若只在流中途断开，可支持 Resume Cursor，或重新生成并明确结果可能不同。

## Markdown 与安全

Sanitize Rendered Markdown/HTML，外链设置安全属性，Code Block 做长度限制。不要执行模型生成的 Script。Citation 链接由 Backend 验证和生成。

## 错误体验

区分 Rate Limit、Network、Safety Refusal、Tool Failure 和 Server Error。提供可操作 Retry，而不是只显示“Something went wrong”。

## Accessibility

Streaming 区域避免每个 Token 触发 Screen Reader；完成段落后用适当 Live Region 通知。输入、Stop、Retry 支持 Keyboard 和 Focus 管理。

## 总结

聊天 UI 是一个异步状态机。稳定 Streaming Protocol、批量渲染、Cancellation、幂等 Retry 和安全 Markdown 决定生产体验。
