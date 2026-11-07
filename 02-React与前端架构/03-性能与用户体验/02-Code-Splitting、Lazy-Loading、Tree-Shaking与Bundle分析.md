# 02. Code Splitting、Lazy Loading、Tree Shaking 与 Bundle 分析

## JavaScript 成本

Bundle 不只消耗下载流量，还需要 Parse、Compile 和 Execute。低端 Mobile Device 上，执行成本常比下载更严重。

## Code Splitting

按 Route 是最常见边界：

```tsx
const AdminPage = lazy(() => import('./AdminPage'));
```

还可按重型 Feature 拆分，如 Chart、Editor、PDF Viewer。切分过细会增加请求、Loading Boundary 和 Cache 管理成本。

## Lazy Loading

只在需要时加载资源，但用户点击后才开始下载可能造成等待。对高概率下一步可使用 Prefetch/Preload，但要避免浪费 Mobile Bandwidth。

优先级：

- 当前页面关键资源：立即加载或 Preload。
- 高概率下一 Route：空闲时 Prefetch。
- 低概率高级 Feature：交互后加载。

## Tree Shaking

依赖 ES Module 静态结构移除未使用 Export。常见阻碍：

- CommonJS。
- Module 顶层 Side Effect。
- 错误的 `sideEffects` 配置。
- 导入整个 Library。
- 运行时动态访问 Export。

Tree Shaking 不等于自动删除所有第三方未用代码，需要检查 Build Output。

## Bundle Analysis

关注：

- Initial JS 与 Route Chunk。
- 重复 Dependency。
- 多版本 Library。
- 巨大 Locale/Data。
- Source Map 对应的真实模块。
- Gzip/Brotli 后大小与未压缩执行成本。

发现 Regression 后设置 CI Budget，例如 Initial JS 超过阈值则阻止 Merge 或要求批准。

## 常用优化

- 替换或按需导入重型 Library。
- Server Component 减少 Client JS。
- 删除不必要 Polyfill。
- 延迟 Analytics/Third-party Script。
- 合理共享 Vendor Chunk，避免缓存失效扩大。
- 长期 Hash Cache 静态 Chunk。

## 常见错误

- 只看压缩大小，不看执行时间。
- 所有 Component 都 Lazy，产生 Loading 碎片。
- Prefetch 整个应用，失去切分意义。
- 把 Common Chunk 做得过大，一次改动导致所有用户重下。

## 总结

性能目标不是 Chunk 越多越好，而是让用户只在需要时支付必要的 Network 和 Main-thread Cost，并持续通过分析和 Budget 防止退化。
