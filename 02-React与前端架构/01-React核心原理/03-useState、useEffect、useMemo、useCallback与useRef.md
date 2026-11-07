# 03. useState、useEffect、useMemo、useCallback 与 useRef

## useState

`useState` 保存影响 UI 的本地状态。更新是排队和批处理的，不应立即读取并假设值已变化。

```tsx
setItems(current => [...current, newItem]);
```

State 应保持最小化。能够从 Props 或其他 State 计算出的值不应重复存储，否则容易不同步。

## useEffect

`useEffect` 用于与 React 外部系统同步，例如 Network、Browser API、Subscription。它不是通用“Render 后执行代码”的容器。

```tsx
useEffect(() => {
  const controller = new AbortController();
  loadUser(userId, controller.signal).then(setUser);
  return () => controller.abort();
}, [userId]);
```

生产应用中的 Server State 更适合交给 React Query 等 Library，处理 Cache、Deduplication、Retry 和 Race Condition。

## useMemo 与 useCallback

- `useMemo` 缓存计算结果。
- `useCallback` 缓存函数引用。

它们是性能优化，不是语义保证。适用条件通常是计算昂贵，或稳定引用能避免已 `memo` 的 Child 重渲染。

过度使用会增加 Dependency 管理和内存成本。先通过 Profiler 确认瓶颈。

## useRef

`useRef` 保存跨 Render 的可变值，修改 `.current` 不触发 Render。用途包括 DOM Ref、Timer ID、Imperative Integration 和不影响 UI 的最新值。

```tsx
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus();
```

若数据变化应该更新 UI，就应使用 State，而不是 Ref。

## 选择方法

| 需求 | 工具 |
| --- | --- |
| 影响 UI 的本地数据 | `useState` |
| 同步外部系统 | `useEffect` |
| 昂贵派生计算 | `useMemo` |
| 稳定函数引用 | `useCallback` |
| 不触发 Render 的可变值 | `useRef` |

## 面试陷阱

- `useMemo` 不保证业务正确性。
- 空 Dependency Array 不等于 Component 真正生命周期只执行一次。
- Ref 不是绕过 React State Model 的万能方式。
- Effect 中的异步响应可能乱序，必须取消或忽略旧请求。

## 自测题

1. 什么数据不应该放入 State？
2. `useCallback` 在什么情况下没有收益？
3. State 与 Ref 的决定性区别是什么？

## 总结

先用 State 和 Pure Render 建立正确模型，只为外部同步使用 Effect，只在测量后加入 Memoization，并用 Ref 保存不参与 UI 的可变数据。
