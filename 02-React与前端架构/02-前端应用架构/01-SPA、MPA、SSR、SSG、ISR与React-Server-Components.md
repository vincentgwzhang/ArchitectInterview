# 01. SPA、MPA、SSR、SSG、ISR 与 React Server Components

## 渲染模式

### SPA

Browser 下载 JavaScript 后在 Client Render 和路由。适合后台系统和强交互应用，但首屏、SEO、Bundle 与低端设备性能需要特别治理。

### MPA

每次 Navigation 从 Server 获取新 HTML。边界清晰、渐进增强容易，但跨页面状态和应用式交互较弱。

### SSR

每次请求在 Server 生成 HTML，再在 Browser Hydrate。适合动态、个性化且重视首屏/SEO 的页面。代价是 Server Cost、Cache 复杂度和 Hydration。

### SSG

Build Time 生成静态 HTML，适合 Documentation、Marketing 和变化较少内容。发布后内容更新通常需要重建。

### ISR

静态页面按时间或事件增量再生成，在 Freshness 与 CDN Performance 之间折中。需理解 Stale Window 和 Cache Invalidation。

### React Server Components（RSC）

RSC 在 Server 执行，不把 Component JavaScript 发送到 Client，可直接访问 Server Resource，并与 Client Component 组合。它不等同于 SSR：RSC 决定 Component 在哪里执行，SSR 决定如何生成初始 HTML。

## 选择矩阵

| 场景 | 常见选择 |
| --- | --- |
| 内部 Dashboard | SPA/CSR |
| Marketing Site | SSG |
| 新闻和商品详情 | SSR/ISR |
| 高度个性化首屏 | SSR |
| 数据密集且需减少 Client JS | RSC + SSR/Streaming |

同一应用可以按 Route 混合策略，不需要全局只选一种。

## 架构问题

- SEO 和 Social Preview 是否重要？
- 内容 Freshness 要求是什么？
- 是否包含用户级敏感数据，能否 CDN Cache？
- Server Render 的 Region、Latency 和 Capacity 如何？
- Hydration Payload 多大？
- Client Navigation 是否需要 Offline 能力？

## 常见错误

- 为内部后台系统盲目使用 SSR。
- SSR Response 缓存时混入用户数据。
- 把 SSG 当成永远不会过期的内容。
- RSC 与 Client Component 边界不清，导致 Serialization 或 Bundle 问题。

## 面试回答

先说明 User、SEO、Freshness、Personalization、Interactivity 与 Cost，再选择 Route-level Rendering Strategy，并说明 Cache、Hydration 和 Failure Behavior。

## 总结

渲染模式是业务和运行时权衡。现代 React 应用通常采用 Hybrid Rendering，让每类页面使用最符合其数据与体验要求的策略。
