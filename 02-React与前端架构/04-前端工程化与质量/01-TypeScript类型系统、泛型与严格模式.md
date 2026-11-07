# 01. TypeScript 类型系统、泛型与严格模式

## 类型的价值

TypeScript 在 Compile Time 描述程序约束，改善 Refactoring、API Contract 和 IDE Feedback。类型不会自动验证 Runtime Input，外部 JSON 仍需 Schema Validation。

## 严格模式

建议开启 `strict`，尤其关注：

- `strictNullChecks`。
- `noImplicitAny`。
- `noUncheckedIndexedAccess`。
- `exactOptionalPropertyTypes`。

迁移遗留项目可分阶段开启，但新代码不应依赖广泛 `any`。

## Union 与 Narrowing

用 Discriminated Union 表达状态机：

```ts
type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };
```

它比多个互相矛盾的 Boolean 更能防止非法状态。

使用 `never` 做 Exhaustive Check，新增状态时让 Compiler 提醒所有处理位置。

## 泛型

泛型表达输入与输出的关系，而不是逃避具体类型：

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}
```

避免复杂到无法阅读的类型体操。Public API 的类型应清晰、稳定并提供良好 Error Message。

## unknown、any 与类型断言

- `unknown`：必须 Narrow 后使用，适合外部输入。
- `any`：关闭类型检查，只在受控迁移边界使用。
- `as`：告诉 Compiler 相信开发者，不产生 Runtime Check。

通过 Zod 等 Runtime Schema 验证 API Response，再推导 Type，避免“类型说安全但数据不安全”。

## React 类型设计

- Props 表达必需、可选和互斥关系。
- Event 使用准确类型。
- `children` 不应默认存在，按 Contract 定义。
- 避免把所有 Props 做成 Optional。
- Domain ID 可用 Branded Type 降低混用。

## 架构治理

共享类型不应让 Frontend 直接依赖 Backend 内部 Entity。可以从 OpenAPI/GraphQL 生成 Transport Type，再转换为 UI Domain Model。

## 总结

TypeScript 的目标是让非法状态难以表示，并让边界契约可验证。严格模式、Narrowing、适度泛型和 Runtime Validation 应一起使用。
