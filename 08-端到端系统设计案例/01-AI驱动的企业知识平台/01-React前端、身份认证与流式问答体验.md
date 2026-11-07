# 01. React 前端、身份认证与流式问答体验

## 用户流程

企业用户 SSO 登录，选择知识范围，提问并流式查看答案、Citation 和 Tool 状态。支持 Stop、Retry、Feedback、历史记录与来源预览。

## 前端架构

React Route/Feature 分层；Server State 用 Query Cache；会话使用显式 Message State Machine。SSE/Fetch Stream 解析 `delta/citation/completed/error`，用 `AbortController` 取消并批量更新 UI。

## 身份

OIDC Authorization Code + PKCE 或 BFF Session。Backend 从可信 Token 获取 Tenant/Group，前端隐藏按钮不构成授权。Session Cookie 使用 HttpOnly/Secure/SameSite。

## 体验

显示 Retrieval/Generating/Tool 状态；证据不足明确拒答；Citation 可打开并高亮原文。错误区分权限、限流、模型和网络，并提供可恢复路径。

## 安全与可访问性

Sanitize Markdown，外链安全，禁止执行生成代码。Streaming 对 Screen Reader 降频通知，Keyboard 可操作 Stop/Retry/Source。

## 指标

TTFT、完成时间、JS Error、Citation Click、Retry、Task Completion 和 Feedback，关联 Request/Trace/Release。

## 总结

AI Chat 前端是安全的异步状态机，不只是文本框；身份、流协议、Citation 和失败体验必须一起设计。
