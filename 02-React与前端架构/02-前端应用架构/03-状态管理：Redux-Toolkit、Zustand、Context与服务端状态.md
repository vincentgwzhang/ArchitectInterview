# 03. 状态管理：Redux Toolkit、Zustand、Context 与服务端状态

## 先分类状态

- Local UI State：Modal、Input、Tab。
- URL State：Filter、Page、Selected ID。
- Server State：Backend Data、Cache、Loading、Staleness。
- Global Client State：跨页面 Workflow、Draft、权限上下文。
- Form State：Value、Validation、Touched。

不要用同一个 Store 管理所有类型。

## Context

适合 Theme、Locale、Auth Context 和低频依赖注入。它没有 Selector 和完整状态治理，高频大对象更新会扩大 Render 范围。

## Redux Toolkit

适合复杂、可预测的 Global Client State：

- 多 Feature 共享。
- 需要明确 Event/Action。
- 复杂更新和 Middleware。
- 强 DevTools、Time-travel 或审计需求。

使用 Slice、Immer 和 Memoized Selector。不要把所有 API Response 手工复制进 Slice；RTK Query 或 React Query 更适合 Server State。

## Zustand

API 简洁、Selector 细粒度，适合中等复杂度 Client Store。需要团队自行定义边界、Action Convention、Persistence 和 Debug 规范，避免形成无结构 Global Mutable Store。

## Server State

Server State 具有 Remote Ownership、Staleness、Cache、Retry 和并发请求特征。React Query 等工具提供：

- Query Key。
- Cache 与 Garbage Collection。
- Deduplication。
- Refetch 与 Invalidation。
- Optimistic Update。
- Loading/Error State。

## URL State

可分享、可刷新恢复、支持 Back/Forward 的状态应优先进入 URL，例如 Search Filter 和 Pagination。不要只存 Global Store 造成链接不可重现。

## 决策顺序

1. 能否从现有输入派生？不存。
2. 是否只属于一个 Component？Local State。
3. 是否来自 Server？Query Cache。
4. 是否应该可分享？URL。
5. 是否跨 Feature 且是 Client-owned？Redux/Zustand。

## 常见错误

- Redux 保存每个 Input Value。
- Context 承担高频全局 Store。
- 同一数据同时存在 Query Cache 和 Redux，形成双 Source of Truth。
- Persist 整个 Store，导致版本迁移和敏感数据风险。

## 面试要点

好的回答不是“Redux vs Zustand 谁更好”，而是先分类 State，再按 Ownership、Lifetime、Sharing、Update Frequency 和 Debug Need 选择。

## 总结

状态管理的核心是 Ownership 和生命周期。Local 优先，Server State 交给 Cache Library，可重现状态进入 URL，只有真正跨域的 Client State 才进入 Global Store。
