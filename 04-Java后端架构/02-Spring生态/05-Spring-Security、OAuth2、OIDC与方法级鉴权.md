# 05. Spring Security、OAuth 2.0、OIDC 与方法级鉴权

## Authentication 与 Authorization

Authentication 确认主体，Authorization 判断主体能否对 Resource 执行动作。React 隐藏按钮不是授权，Backend 必须强制执行。

## OAuth 2.0 与 OIDC

- OAuth 2.0：Delegated Authorization Framework。
- OIDC：在 OAuth 2.0 上增加 Identity Layer 和 ID Token。

常见 Web 使用 Authorization Code + PKCE。Service-to-service 使用 Client Credentials 或 Workload Identity。不要使用已不推荐的 Implicit Flow。

## Token

Access Token 面向 Resource Server；ID Token 面向 Client，不应用作通用 API Access Token。验证 Issuer、Audience、Signature、Expiry 和 Scope。

JWT 自包含、验证快，但撤销困难；Opaque Token 可集中 Introspection。选择取决于延迟、撤销和治理要求。

## Spring Security

Security Filter Chain 处理请求级安全；方法级 `@PreAuthorize` 适合靠近业务操作的规则：

```java
@PreAuthorize("hasAuthority('order:read') and @access.canRead(#orderId)")
```

只检查 Role 不足以防 IDOR，必须验证 Resource Ownership/Tenant。

## CSRF 与 Session

Cookie Session 需要 CSRF 防护；Bearer Header 的风险模型不同，但 XSS 仍可窃取 Token。Web 应优先考虑 HttpOnly、Secure、SameSite Cookie 与 BFF 架构。

## 运营

最小 Scope、短期 Credential、Key Rotation、Audit、Break-glass 和授权失败 Metric。不要在 Log 输出 Token。

## 总结

安全架构将身份、委托、Scope 和 Resource Policy 分层。Spring Security 提供机制，业务授权仍需明确 Domain Rule。
