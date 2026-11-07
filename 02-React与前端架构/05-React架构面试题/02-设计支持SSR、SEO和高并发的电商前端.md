# 02. 设计支持 SSR、SEO 和高并发的电商前端

## 目标

- 商品与分类页可被搜索引擎索引。
- 首屏快速，支持大促流量。
- 价格、库存和个性化保持正确。
- Checkout 安全且高可用。

## 渲染策略

- Marketing/静态内容：SSG。
- Category/Product：ISR 或 Cacheable SSR。
- Search：SSR 首屏 + Client Navigation。
- Cart/Account/Checkout：动态 SSR 或 Client Render，禁止共享私有 Cache。

使用 React Streaming 和 Suspense 先输出 Shell 与关键商品信息，推荐等非关键区域稍后到达。

## CDN 与 Cache

静态 Asset 使用 Hash + Immutable。公开商品页以 Product/Locale/Currency 设计 Cache Key，并通过 Event Revalidation 更新。价格和库存可显示 Snapshot，但 Checkout 必须由 Backend 重新验证。

## SEO

- Server 输出有效 Title、Description、Canonical 和 Structured Data。
- Stable URL、Sitemap、Robots。
- 正确 Status Code，软 404 不可取。
- Pagination 与 Filter 控制重复内容。

## 高并发

CDN 吸收公开读取，Origin Multi-AZ Auto Scaling。Backend 使用 Cache、Read Model 和保护机制。大促时对 Recommendation 降级，不阻塞 Product/Checkout。

## 前端性能

- Hero Image 优先并响应式。
- Route Code Splitting。
- 控制 Third-party Script。
- RUM Core Web Vitals。
- Server Component 减少 Client JS。

## 安全

HttpOnly Session、CSRF 防护、CSP、Backend Authorization、支付 Tokenization。SSR Cache 严格区分公开与用户响应。

## Failure UX

库存/推荐局部 Error Boundary，Cart 操作 Idempotent，支付超时显示 Pending 并查询最终状态，避免用户重复付款。

## 面试亮点

讨论 Cache Freshness 与正确性、公开/私有数据隔离、Peak Degradation、SEO 可爬取性和 Checkout 最终验证。
