# 01. WebSocket/SSE、连接管理与消息路由

## 协议

SSE 适合 Server→Browser 单向通知；WebSocket 适合双向聊天/实时协作；Mobile Push 通过 APNs/FCM。协议按方向、连接时长和网络环境选择。

## Connection Gateway

Gateway 处理 Auth、Heartbeat、Rate、Backpressure 和 Connection ID，将 `userId/deviceId -> gateway/shard` 存入有 TTL 的 Registry。

## 路由

Producer 发布 Notification Event；Router 根据 User/Topic/Preference 找 Channel；实时连接通过 Pub/Sub/Partition 送到对应 Gateway，离线进入 Push/Email Queue。

## 可靠性

连接断开自动重连，Client 带 Last Seen Cursor；Server 可重放有限窗口。Heartbeat 检测半开连接；慢 Client 有 Buffer 上限和 Drop/Disconnect Policy。

## Scale

连接数、Message/sec、Fan-out、Payload 和 Region 是核心容量。Gateway 无状态化到可恢复 Registry，跨 AZ，避免全局 Sticky State。

## 总结

实时平台要同时管理长连接、消息持久性和离线通道。连接本身不是可靠交付保证。
