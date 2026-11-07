# 05. React 性能、CI/CD、安全与业务监控

## React

商品页 SSR/ISR + CDN，Checkout 动态私有页面；Route Split、响应式图片、控制 Third-party Script。RUM 监控 LCP/INP/CLS 与 Conversion。

## CI/CD

Trunk-based、Preview、Contract/E2E、Bundle/Accessibility/Security Gate。Canary 按 Region/Tenant，Feature Flag 控制业务发布；数据库 Expand-and-Contract。

## 安全

OIDC/BFF Session、CSRF/CSP、WAF/Rate Limit、Backend Resource Authorization、Payment Tokenization、Secret/PII 不入 Log。

## 业务监控

Browse → Product → Cart → Checkout → Payment → Order Funnel，配合 API SLO、Payment Pending、库存冲突和 Cost/order。Release Version 关联指标。

## 失败体验

局部 Error Boundary；支付未知状态不允许重复提交；Cart/Order 可恢复。Accessibility 和 Mobile Low-end Device 纳入测试。

## 总结

电商体验、交付与安全必须共同服务交易成功率；技术指标要连接业务 Funnel 才能指导优先级。
