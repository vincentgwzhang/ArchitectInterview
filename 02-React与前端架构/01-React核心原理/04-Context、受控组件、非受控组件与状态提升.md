# 04. Context、受控组件、非受控组件与状态提升

## 状态所有权

每份 State 应有明确的 Single Source of Truth。把状态放在需要共同读取或修改它们的最近公共父 Component，称为 Lifting State Up。

如果状态提升过高，会导致 Prop Drilling、过大重渲染范围和难以理解的 Ownership。优先让 State Local，只在确有共享需求时提升。

## 受控组件

受控表单的值由 React State 驱动：

```tsx
<input value={email} onChange={e => setEmail(e.target.value)} />
```

优点：Validation、条件 UI 和提交状态容易统一。代价是每次输入触发 State Update，大型表单需要合理拆分或使用表单 Library。

## 非受控组件

非受控表单让 DOM 保存当前值，通过 `ref` 或 FormData 读取：

```tsx
<input name="email" defaultValue="" />
```

适合简单表单、文件输入、与非 React Widget 集成。不要同时混用 `value` 和 `defaultValue` 改变控制模式。

## Context

Context 解决跨层传递稳定的共享依赖，例如 Theme、Locale、Authenticated User 或 Service Instance。

```tsx
const AuthContext = createContext<AuthContextValue | null>(null);
```

Provider 的 `value` 改变时，读取该 Context 的 Consumer 会重渲染。高频、大对象状态放入单一 Context 可能造成性能与耦合问题。

改进方法：

- 按变化频率和职责拆分 Context。
- 稳定 Provider Value。
- Server State 使用 Query Cache。
- 复杂 Client State 使用专门 Store 和 Selector。

## 决策指南

1. 只在一个 Component 使用：Local State。
2. 兄弟 Component 共享：提升到最近父级。
3. 跨多层传递稳定依赖：Context。
4. 高频全局状态并需细粒度订阅：State Store。
5. 来自 Backend 的可缓存数据：Server-state Library。

## 常见错误

- 把 Context 当 Redux 替代品并存入全部状态。
- 在多个位置复制同一 State。
- State 提升到 Application Root，只为避免两层 Props。
- 表单 State、Server State 和 URL State 混为一谈。

## 自测题

1. 什么是 Single Source of Truth？
2. Context 更新为什么可能造成大范围 Render？
3. 哪些表单更适合非受控模式？

## 总结

状态架构首先是 Ownership 设计。Local 优先、按需提升、Context 传递稳定跨层依赖，并根据状态类别选择合适工具。
