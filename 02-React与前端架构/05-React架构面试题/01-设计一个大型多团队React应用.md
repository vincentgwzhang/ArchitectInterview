# 01. 设计一个大型多团队 React 应用

## 需求澄清

先问：团队数量、Domain、用户规模、发布频率、SEO、独立部署需求、共享品牌、遗留系统与 NFR。

## 推荐基线

如果没有明确独立部署需求，优先 Monorepo + Modular Frontend：

```text
apps/web
packages/design-system
packages/platform
features/catalog
features/checkout
features/account
```

每个 Feature 有 Owner、Public API 和禁止跨边界的 Lint Rule。Platform 提供 Router、Auth、Telemetry、API Client、Feature Flag 和 CI Template。

## 状态与数据

- Local UI State 保持局部。
- Server State 使用 Query Cache。
- Filter/Pagination 进入 URL。
- 跨 Feature Client State 使用明确 Store Slice。
- 不共享 Backend 内部 Entity。

## 交付与治理

- Trunk-based Development 与 Preview Environment。
- Type Check、Lint、Component/Contract/E2E Test。
- Bundle 与 Core Web Vitals Budget。
- Design System 和 Accessibility Gate。
- CODEOWNERS 与风险型 Review。

## 独立部署

当多个团队真正需要独立 Release 时，按 Route/Domain 引入 Micro-frontend。Shell 管理公共体验，Remote 有 Error Boundary、Version Contract 和 Fallback。

## Observability

RUM、Error Tracking、Release Version、Trace Correlation 和 Business Funnel 必须统一，但 Dashboard 按 Team/Feature 分解。

## 演进路线

1. 建立 Module Boundary。
2. 提取 Design System 与 Platform API。
3. 自动化 Dependency Rule。
4. 测量发布耦合。
5. 仅拆分需要独立交付的 Domain。

## 面试亮点

说明为何不是默认 Micro-frontend；讨论 Ownership、Contract、Developer Experience、Runtime Failure、Performance 和组织结构，而不只画 Component。
