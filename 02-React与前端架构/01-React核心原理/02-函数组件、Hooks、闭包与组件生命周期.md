# 02. 函数组件、Hooks、闭包与组件生命周期

## 学习目标

- 理解 Function Component 每次 Render 都是一次独立执行。
- 解释 Hooks、Closure 和 Lifecycle 的关系。
- 识别 Stale Closure 与 Effect Cleanup 问题。

## Render Snapshot

每次 Render 都创建一组新的 Props、State 和函数闭包。Event Handler 读取的是创建它那次 Render 的 Snapshot，而不是自动读取未来值。

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const incrementLater = () => {
    setTimeout(() => setCount(current => current + 1), 1000);
  };
  return <button onClick={incrementLater}>{count}</button>;
}
```

Functional Update 避免闭包中的旧值。若写成 `setCount(count + 1)`，多个延迟操作可能基于同一个旧 Snapshot。

## Hooks 规则

Hooks 必须在 Component 或 Custom Hook 顶层以稳定顺序调用。React 依靠调用顺序把每个 Hook 与内部 State Slot 对应，因此不能放在条件分支或普通循环中。

Custom Hook 复用的是 Stateful Logic，不共享 State 本身。每次调用都拥有独立状态。

## Lifecycle 思维

Function Component 更适合用“同步某个外部系统”理解 Effect：

```tsx
useEffect(() => {
  const connection = connect(roomId);
  return () => connection.disconnect();
}, [roomId]);
```

当 `roomId` 改变时，React 先清理旧同步，再建立新同步。Cleanup 也在 Unmount 时运行。Development Strict Mode 可能额外执行 Setup/Cleanup，用于发现不安全 Side Effect。

## 常见问题

- Effect 缺少 Dependency，读取旧 Props/State。
- 用 Effect 计算可在 Render 中派生的数据。
- Cleanup 缺失导致 Listener、Timer 或 Request 泄漏。
- 在 Render 中修改外部 Store。
- 把 Class Lifecycle 生硬映射为一组空数组 Effect。

## 面试要点

解释“每个 Render 都有自己的 Closure”，再说明 Dependency Array 不是随意性能开关，而是 Effect 使用的 Reactive Value 声明。

## 自测题

1. 为什么 Hooks 不能条件调用？
2. Stale Closure 是如何产生的？
3. Strict Mode 的重复 Effect 为什么有价值？

## 总结

Function Component 是从输入到 UI 的 Pure Calculation。Hooks 为每次 Render Snapshot 提供 State 和 Effect 能力；理解 Closure 是正确使用 React 的关键。
