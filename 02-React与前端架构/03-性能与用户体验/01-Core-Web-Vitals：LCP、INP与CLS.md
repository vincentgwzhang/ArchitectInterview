# 01. Core Web Vitals：LCP、INP 与 CLS

## 三个指标

- LCP（Largest Contentful Paint）：主要内容何时可见，衡量加载体验。
- INP（Interaction to Next Paint）：用户交互到下一次 Paint 的响应延迟，衡量交互体验。
- CLS（Cumulative Layout Shift）：非预期布局移动，衡量视觉稳定性。

评估应使用真实用户的 75th Percentile，并按 Mobile/Desktop、Route、Region 和 Device 分析。

## 优化 LCP

先识别 LCP Element，常见瓶颈是 TTFB、资源发现、下载和 Render Delay。

- 使用 CDN、Cache 和 Streaming 改善 TTFB。
- Hero Image 使用正确尺寸、现代格式和高优先级加载。
- 不要 Lazy Load 首屏 LCP Image。
- 减少 Render-blocking CSS/JS。
- SSR/SSG 提供早期 HTML。
- 控制 Client Bundle 和 Hydration。

## 优化 INP

- 拆分 Long Task。
- 减少一次交互触发的 Render 范围。
- 使用 Transition 延后非紧急更新。
- Virtualize 大列表。
- 将重 CPU 工作移到 Web Worker。
- 避免同步读取/写入 DOM 造成 Layout Thrashing。

INP 问题需要通过 Performance Profile 找到 Main Thread 阻塞，而不只是增加 `useMemo`。

## 优化 CLS

- 图片和 Video 设置宽高或 `aspect-ratio`。
- 为异步内容和广告预留空间。
- 避免在现有内容上方动态插入 Banner。
- 正确处理 Web Font，减少字体切换位移。
- Skeleton 尺寸应接近真实内容。

## Lab 与 Field Data

- Lab：Lighthouse、Performance Panel，适合开发和回归。
- Field/RUM：真实设备、网络和用户，反映生产体验。

二者都需要。Lab 稳定可重复，Field 能发现长尾设备和真实交互。

## Performance Budget

为 Route 定义 JS、Image、LCP、INP 和 CLS Budget，并在 CI 与 RUM 中监控。指标必须关联 Release Version，才能发现 Regression。

## 面试要点

不要只背目标数值。说明如何从 RUM 分段定位、识别主导因素、设 Budget、验证改善，并防止回归。

## 总结

Core Web Vitals 是用户体验结果指标。优化需要连接 Server、Network、Asset、JavaScript、React Render 和页面布局，而不是单点技巧。
