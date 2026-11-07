# 05. Error Boundary、Suspense、并发渲染与流式渲染

## Error Boundary

Error Boundary 捕获其 Child Tree 在 Render、Lifecycle 和 Constructor 中抛出的错误，并展示 Fallback UI。它通常不捕获 Event Handler、异步 Callback 或自身错误。

边界应围绕可独立失败的 UI 区域：Route、Widget、Editor，而不是全站只放一个边界。记录 Error、Component Stack、Route、Version 和 Trace 信息。

```tsx
<ErrorBoundary fallback={<PanelError />}>
  <RecommendationPanel />
</ErrorBoundary>
```

## Suspense

Suspense 用声明方式定义 Child 尚未准备好时的 Loading Boundary：

```tsx
<Suspense fallback={<PageSkeleton />}>
  <ProductPage />
</Suspense>
```

它可用于 Code Splitting，以及 Framework 支持的数据加载。边界粒度决定用户体验：过大导致整页闪烁，过细导致多个 Spinner 和 Layout Shift。

## Concurrent Rendering

并发渲染允许 React 暂停低优先级 Render，先响应输入等高优先级更新。`startTransition` 可标记非紧急更新：

```tsx
startTransition(() => setSearchQuery(nextQuery));
```

Render 可能被重启或丢弃，因此必须 Pure。不要依赖“每次 Render 一定 Commit”。

## 流式渲染

SSR Streaming 允许 Server 先发送 Shell，再随着 Suspense Boundary 准备完成逐段输出 HTML，降低 Time to First Byte 和首屏等待。

配合 Selective Hydration，Browser 可以优先 Hydrate 用户正在交互的区域。架构上需要考虑：

- Server/Client Component Boundary。
- Loading 与 Error UI。
- Cache 和用户隔离。
- Hydration Mismatch。
- CDN/Proxy 是否支持 Streaming。

## Failure UX

加载、空数据、部分错误和完全错误是不同状态。优秀界面应允许局部 Retry，并保留未失败区域。Error Boundary 不是吞掉错误，必须连接 Observability。

## 面试要点

- Suspense 是协调“未准备好”的 UI Boundary，不等同于通用 Fetch Library。
- Concurrent Rendering 改善响应性，不等于所有更新更快。
- Error Boundary 与 Backend Resilience 一样，应按 Failure Domain 划分。

## 自测题

1. Error Boundary 不会捕获哪些错误？
2. 为什么 Concurrent Render 要求 Component Pure？
3. Suspense Boundary 过细会造成什么体验问题？

## 总结

通过 Error Boundary 隔离失败、Suspense 协调等待、Transition 区分优先级、Streaming 渐进交付内容，可以构建更具 Resilience 的 React UX。
