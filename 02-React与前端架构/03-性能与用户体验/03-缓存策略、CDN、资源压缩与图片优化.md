# 03. 缓存策略、CDN、资源压缩与图片优化

## Cache 层次

```text
Browser Cache -> CDN/Edge -> Reverse Proxy -> Application Cache -> Database
```

每层应有明确 Ownership、TTL、Invalidation 和 Privacy Policy。

## 静态资源

带 Content Hash 的 JS/CSS/Image 可以使用：

```http
Cache-Control: public, max-age=31536000, immutable
```

HTML 通常使用较短 TTL 或 `no-cache`，确保能发现新 Asset Hash。不要给无版本文件设置永久缓存。

## API/页面缓存

理解：

- `Cache-Control`。
- `ETag` / `If-None-Match`。
- `Last-Modified`。
- `s-maxage`。
- `stale-while-revalidate`。
- `Vary`。

用户私有响应不能进入共享 CDN Cache。Authorization、Cookie、Locale 和 Experiment 都可能影响 Cache Key，Key 设计错误会导致数据泄漏或低 Hit Ratio。

## CDN

CDN 降低距离延迟并吸收流量。需要考虑：

- Origin Protection。
- Cache Key Normalization。
- Purge/Invalidation。
- Signed URL/Cookie。
- Regional Regulation。
- Origin Failover。
- Hit Ratio 与 Egress Cost。

## 压缩

Text 使用 Brotli/Gzip；已经压缩的 JPEG/Video 再 Gzip 收益有限。避免 Runtime 重复压缩耗 CPU，可在 Build 或 CDN 预生成版本。

## 图片优化

- 使用 AVIF/WebP 并提供 Fallback。
- 通过 `srcset`/`sizes` 响应式选择尺寸。
- 明确 width/height 防止 CLS。
- 首屏 LCP Image 高优先级，非首屏 Lazy Load。
- 使用 Image CDN 动态 Resize。
- 限制 Upload 尺寸并生成 Thumbnail。

## Invalidation

“缓存失效很难”的根本原因是多个 Source of Truth。优先使用版本化 URL；业务数据可通过短 TTL、Event Invalidation 或 Revalidation。必须定义缓存错误时的最大 Staleness。

## 监控

观察 CDN Hit Ratio、Origin RPS、TTFB、Egress、Purge Error 和不同资源大小。优化前后同时看用户性能与成本。

## 总结

缓存是数据正确性设计。通过版本化静态资源、安全 Cache Key、明确 Staleness 和图片 Pipeline，才能同时改善速度、可靠性与成本。
