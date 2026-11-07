# 02. 组件分层、Feature-based 目录与领域边界

## 目标

前端架构需要让 Feature 容易定位、修改、测试和删除。目录结构应表达业务边界，而不只是文件类型。

## Component 层次

可以区分：

- UI Primitive：Button、Input、Modal。
- Composite Component：AddressForm、ProductCard。
- Feature Component：Checkout、SearchFilters。
- Route/Page：数据组合、权限和页面布局。
- Application Shell：Router、Provider、全局 Error Boundary。

Primitive 不应依赖具体业务；Feature 可以使用 Design System 并承载业务交互。

## Feature-based 结构

```text
src/
├── app/                 # bootstrap, router, providers
├── features/
│   ├── checkout/
│   │   ├── api/
│   │   ├── components/
│   │   ├── model/
│   │   └── index.ts
│   └── catalog/
├── shared/
│   ├── ui/
│   └── infrastructure/
└── pages/
```

每个 Feature 通过 Public `index.ts` 暴露少量 API，其他 Feature 不应导入其内部文件。

## Domain Boundary

前端同样需要 Ubiquitous Language 和 Ownership。`Order`、`Cart`、`Payment` 不应变成一个巨大 Global Model。Backend DTO 也不应直接扩散到所有 UI 层，可在 API Adapter 中转换为 Frontend Model。

## Dependency Rule

```text
app/pages -> features -> shared
```

避免 `shared` 反向依赖 Feature，避免 Feature 循环依赖。可用 ESLint Boundary Rule 自动检查。

## Smart 与 Presentational

不必机械区分所有 Component，但应让数据获取、业务编排与纯 UI 有合理边界。纯 UI Component 接收明确 Props，容易在 Storybook 和测试中复用。

## 何时拆 Package

Monorepo 中独立 Package 适合：

- 多应用共享 Design System。
- 独立 Ownership 和 Release。
- 清晰稳定的 Public Contract。

不要把每个 Feature 都拆成 Package，否则 Build、Version 和 Dependency 管理成本上升。

## 常见反模式

- 巨大的 `components/`、`utils/` 和 `services/`。
- `shared` 成为没有 Ownership 的垃圾场。
- 通过深层相对路径访问另一个 Feature 内部。
- UI 直接依赖 Backend 生成的所有字段。
- 全局 State 把 Domain Boundary 重新耦合。

## 总结

以 Feature 和 Domain 组织代码，定义 Public API 与单向依赖，既能支持 Modular Frontend，也为未来团队拆分或 Micro-frontend 演进提供基础。
