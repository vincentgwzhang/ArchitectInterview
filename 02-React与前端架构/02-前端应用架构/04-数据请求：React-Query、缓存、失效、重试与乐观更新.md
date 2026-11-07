# 04. 数据请求：React Query、缓存、失效、重试与乐观更新

## Query Key

Query Key 是 Server State 的缓存身份，必须包含影响结果的所有参数：

```tsx
useQuery({
  queryKey: ['orders', { customerId, status, page }],
  queryFn: () => getOrders({ customerId, status, page })
});
```

Key 应稳定、可序列化，并按 Domain 统一 Factory。

## Freshness 与生命周期

- `staleTime`：数据多长时间视为 Fresh。
- Cache/Garbage Collection Time：没有 Consumer 后保留多久。
- Refetch Trigger：Mount、Focus、Reconnect 或手动失效。

不要把 `staleTime: Infinity` 当作性能优化，除非业务上确实不需要刷新。

## Cache Invalidation

Mutation 成功后可以：

- 精确更新 Cache。
- 失效相关 Query 重新获取。
- 使用 Server Response 替换对象。

过宽 Invalidation 导致请求风暴，过窄会显示旧数据。应建立 Query Key 与 Domain Event 的映射。

## Retry

Retry 适用于网络错误和部分 5xx，不适合 4xx Business Error。使用 Exponential Backoff、次数上限，并考虑请求是否 Idempotent。

前端、Gateway 和 Backend 同时 Retry 可能形成乘法放大，应明确每层责任。

## 乐观更新

流程：

1. Cancel 正在进行的相关 Query。
2. Snapshot 旧 Cache。
3. 立即写入预期结果。
4. Mutation 失败时 Rollback。
5. 完成后与 Server Reconcile。

适合点赞、简单状态切换等低冲突操作。支付、库存和复杂权限变更不应假装已经成功。

## Race Condition

- 用户快速切换参数时取消旧 Request。
- Mutation 乱序返回时使用 Version 或重新获取。
- 避免 Component Unmount 后写入无意义 Local State。
- Server 应提供 ETag/Version 支持 Optimistic Concurrency。

## SSR 与 Hydration

Server Prefetch 后 Dehydrate Cache，Client Hydrate 时必须保证 Query Key、权限和用户隔离正确。绝不能把一个用户的私有 Cache 响应复用给另一个用户。

## Observability

跟踪 API Error、Latency、Cache Hit、Retry Count 和关键 Mutation Failure。用户看到的 Query Error 应带 Retry 或恢复路径。

## 总结

Server State 管理不仅是 Fetch。稳定 Query Key、合理 Freshness、精确 Invalidation、安全 Retry 和冲突意识共同决定数据正确性与体验。
