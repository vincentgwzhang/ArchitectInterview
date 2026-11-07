# 05. 前端安全：XSS、CSRF、CSP、Token 存储与依赖安全

## XSS

React 默认 Escape 插值内容，但以下场景仍危险：

- `dangerouslySetInnerHTML`。
- 未验证的 URL。
- 第三方 Widget。
- DOM API 直接注入。
- 服务端模板和 Markdown Render。

需要展示 HTML 时使用成熟 Sanitizer，并配置允许的 Tag/Attribute。不要自行用 Regex 清理 HTML。

## CSP

Content Security Policy 限制 Script、Style、Frame 和连接来源。推荐逐步采用 Nonce/Hash，减少 `unsafe-inline`，先 Report-only 观察违规。

CSP 是 Defense in Depth，不替代输出编码和输入处理。

## CSRF

Browser 会自动携带 Cookie，因此攻击站点可能诱导用户发送状态变更请求。防护：

- `SameSite` Cookie。
- CSRF Token。
- 检查 Origin/Referer。
- 状态修改不用 GET。

Authorization Header 不会由 Browser 自动附加，风险模型不同，但仍需防 XSS 窃取 Token。

## Token 存储

长期 Token 放 `localStorage` 会被 XSS 读取。常见 Web 架构使用 `HttpOnly + Secure + SameSite` Cookie，让 JavaScript 无法读取 Session Token，并配合 CSRF 防护。

选择取决于 BFF、Cross-domain、Mobile 与 Identity Flow。不存在脱离 Context 的唯一答案。

## 其他风险

- Clickjacking：`frame-ancestors`/X-Frame-Options。
- Open Redirect：限制 Redirect Target。
- IDOR：Backend 强制 Resource Authorization。
- Sensitive Data：不要放 URL、Log、Analytics、Source Map。
- File Upload：类型、大小、扫描和隔离。

## 依赖安全

- Lockfile 与可信 Registry。
- SCA/Vulnerability Scan。
- 减少依赖和安装 Script。
- 定期升级。
- SBOM、Artifact Provenance 和签名。
- 第三方 Script 使用严格来源与 SRI（适用时）。

## 总结

Browser Security 依赖输出安全、身份会话、CSP、Backend Authorization 和供应链治理的多层防护。前端不能保护 Secret，也不能替代 Server-side Control。
