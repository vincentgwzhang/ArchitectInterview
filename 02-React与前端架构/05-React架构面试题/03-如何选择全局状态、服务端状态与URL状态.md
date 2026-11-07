# 03. 如何选择全局状态、服务端状态与 URL 状态

## 决策框架

对每份状态询问：

1. Source of Truth 在哪里？
2. 谁需要读取和修改？
3. 生命周期多长？
4. 是否应该可分享、Bookmark 和 Back/Forward？
5. 是否存在 Staleness、Cache 和 Retry？
6. 是否包含敏感数据？

## 分类

| 状态 | 示例 | 推荐位置 |
| --- | --- | --- |
| Local UI | Modal、Input | `useState`/Reducer |
| URL | Query、Filter、Page | Router/Search Params |
| Server | User、Order、Product | React Query/RTK Query |
| Global Client | 跨页 Draft、Workflow | Redux/Zustand |
| Context | Theme、Auth Service | Context |
| Form | Validation、Touched | Form Library/Local |

## 典型问题

把 Server Response 复制到 Redux 后，Query Cache 与 Redux 可能同时保存 User，Mutation 后只更新一份，产生双 Source of Truth。

把 Search Filter 只放 Global Store，用户刷新或分享 URL 后状态丢失，也破坏 Browser Navigation。

## Persistence

只持久化有明确业务价值的数据，并定义 Schema Version、Migration、Expiry 与 Logout Cleanup。Token、PII 和 Server Cache 不应随意进入 `localStorage`。

## Derived State

`total = items.reduce(...)` 应在 Render/Selector 中计算，不额外存 State。Memoization 只用于昂贵计算或稳定引用。

## 面试示例

电商 Filter 进入 URL；Product List 在 Query Cache；Cart 如果 Server-owned 也通过 Query 管理；Checkout UI Step 可 Local 或 URL；未提交跨页面 Draft 可进入 Global Client Store。

## 总结

状态工具选择不是规模竞赛。以 Ownership、Sharing、Lifetime、Reproducibility 和 Remote Semantics 分类，能避免大多数状态架构问题。
