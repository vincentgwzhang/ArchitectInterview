# 01. Virtual DOM、Diff、Reconciliation 与 Fiber

## 学习目标

- 理解一次 React Update 如何从 Element 变成 DOM 变化。
- 区分 Virtual DOM、Reconciliation 与 Fiber。
- 解释 `key` 为什么影响正确性和性能。

## 核心原理

React Element 是描述 UI 的普通 JavaScript Object。State 或 Props 变化后，React 重新执行 Component，得到新的 Element Tree，再通过 Reconciliation 比较新旧 Tree，计算最小的 Host Update。

React 的 Diff 基于实用假设：不同 Element Type 通常产生不同 Subtree；开发者通过 `key` 标识同级列表中的稳定身份。因此算法不需要对任意 Tree 做昂贵的全局最优比较。

```tsx
items.map(item => <Row key={item.id} item={item} />)
```

不要在可排序列表中使用 Array Index 作为 `key`。插入、删除或移动后，Component State 可能错误地关联到另一个 Item。

## Fiber

Fiber 是 React 的工作单元和内部 Tree Node。它让 Rendering Work 可以被拆分、设定优先级、中断和恢复。一次 Update 大致分为：

1. Render Phase：计算下一棵 Tree，可被中断，不应产生外部 Side Effect。
2. Commit Phase：同步应用 DOM Change、Ref 和 Layout Effect，不可中断。

Concurrent Rendering 不是让 JavaScript 多线程执行，而是让 React 调度不同优先级的 Render Work，保持交互响应。

## 架构含义

- Render 必须保持 Pure，相同输入应产生相同输出。
- 不要在 Component Body 中请求网络或修改全局状态。
- `useLayoutEffect` 会阻塞 Paint，应谨慎使用。
- 大量 DOM Node、昂贵计算和不稳定 Props 仍会造成性能问题，Virtual DOM 不是自动优化器。

## 面试要点

回答时应说明 Virtual DOM 是 UI 描述，Reconciliation 是比较过程，Fiber 是支持可调度 Rendering 的内部架构。强调 `key` 表示 Identity，而不只是“消除 Warning”。

## 自测题

1. Render Phase 为什么必须无 Side Effect？
2. Component Type 改变时 React 通常如何处理原 Subtree？
3. Index Key 在什么场景下相对安全？

## 总结

React 性能来自可预测的 Diff、批处理和调度，而不是“Virtual DOM 永远比 DOM 快”。正确的 Identity、Pure Render 和合理 Component Boundary 才是基础。
