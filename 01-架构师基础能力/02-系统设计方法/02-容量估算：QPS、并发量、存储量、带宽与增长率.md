# 02. 容量估算：QPS、并发量、存储量、带宽与增长率

## 学习目标

1. 使用 Back-of-the-envelope Calculation 快速估算系统规模。
2. 计算 QPS、并发量、存储、带宽与增长空间。
3. 区分平均值、Peak 值和 Percentile。
4. 用数量级判断架构是否合理，而不是追求虚假精度。

## 一、容量估算的作用

容量估算用于回答：

- 单台应用或数据库是否足够？
- 是否需要 Cache、Queue、Partitioning 或 CDN？
- 系统瓶颈更可能出现在 CPU、Memory、Storage 还是 Network？
- Cloud Cost 大致处于什么数量级？
- 设计需要预留多少增长空间？

估算不是预测未来的精确数字。它的价值是暴露假设、验证数量级并指导设计。

## 二、常用基础数值

```text
1 day = 86,400 seconds，估算时常取 100,000
1 million/day ≈ 12 requests/second
100 million/day ≈ 1,200 requests/second
1 KB × 1 million ≈ 1 GB
1 MB × 1 million ≈ 1 TB
```

面试中可以适当取整，但要说明近似。

## 三、QPS 与 Peak QPS

假设系统有 10 million DAU，每个用户每天执行 20 次读取：

```text
Daily reads = 10,000,000 × 20 = 200,000,000
Average QPS = 200,000,000 / 86,400 ≈ 2,315
```

平均值不能用于直接 Provisioning。流量通常存在日夜波动、营销活动和突发流量。

如果 Peak Factor 取 5：

```text
Peak QPS ≈ 2,315 × 5 ≈ 11,600
```

还需要保留 Headroom，例如 30%：

```text
Design QPS ≈ 11,600 × 1.3 ≈ 15,000
```

### 读写分离

假设读写比例为 100:1：

```text
Peak read QPS = 15,000
Peak write QPS = 150
```

这可能提示读取侧适合 Cache/CDN，而写入侧单个关系型数据库仍可能足够。

## 四、并发量

并发量可以通过 Little's Law 进行粗略估算：

```text
Concurrency = Throughput × Average response time
```

如果系统处理 10,000 RPS，平均响应时间为 200 ms：

```text
Concurrency = 10,000 × 0.2 = 2,000
```

如果是 WebSocket，连接长时间保持，并发量更接近同时在线用户数，而不是短请求公式。

需要区分：

- Request Concurrency。
- Active User Concurrency。
- Open Connection Count。
- Background Job Concurrency。

## 五、应用实例数量

假设 Load Test 显示单个 Java Instance 在目标 Latency 下可处理 500 RPS：

```text
Required instances = 15,000 / 500 = 30
With 30% headroom ≈ 39
```

还要考虑：

- 单个 AZ 故障后的剩余容量。
- Rolling Deployment 时部分实例不可用。
- Auto Scaling 启动时间。
- 下游数据库和连接池上限。

如果部署在 3 个 AZ，并要求丢失一个 AZ 后仍承载全部流量，每个 AZ 不能只部署总容量的三分之一。

## 六、存储估算

基本公式：

```text
Storage = records/day × bytes/record × retention days × replication factor
```

例：每天产生 50 million Event，每条平均 1 KB，保存 365 天：

```text
Raw/day = 50,000,000 × 1 KB ≈ 50 GB
Raw/year ≈ 18.25 TB
Replication factor 3 ≈ 54.75 TB
```

还需要考虑：

- Index 大小。
- Metadata 和 Row Overhead。
- Compression Ratio。
- Backup 与 Snapshot。
- Temporary Data。
- 数据增长率。
- Hot、Warm、Cold Storage 分层。

对数据库，可使用 1.2 到 2 倍系数粗略考虑 Index 和空间开销，但真实值必须通过数据模型和测试确认。

## 七、带宽估算

### 7.1 Egress Bandwidth

如果 Peak 为 5,000 RPS，平均 Response 为 200 KB：

```text
Data/sec = 5,000 × 200 KB = 1,000,000 KB/s ≈ 1 GB/s
Bits/sec ≈ 8 Gbps
```

这提示需要 CDN、压缩和多节点网络能力。

### 7.2 Ingress Bandwidth

如果用户每天上传 1 million 个 10 MB 文件：

```text
Daily ingress = 10 TB
Average ≈ 116 MB/s
```

Peak、Multipart Upload、Retry 和跨 Region Replication 都会增加网络需求与成本。

## 八、Cache 容量

假设需要缓存最近 20 million 个对象，每个序列化后为 2 KB：

```text
Raw cache = 20,000,000 × 2 KB = 40 GB
```

再考虑：

- Key 和数据结构开销。
- Replication。
- Memory Fragmentation。
- Eviction Buffer。
- Failover Capacity。

实际可能需要 80 GB 或更多。Cache Hit Ratio 也必须估算，因为 Miss Traffic 决定后端压力。

```text
Backend read QPS = Total read QPS × (1 - hit ratio)
```

15,000 QPS、95% Hit Ratio 时，Backend 仍需处理 750 QPS。

## 九、数据库连接估算

不要让每个 Instance 无限制建立连接。

假设 40 个 Application Instance，每个 Connection Pool 为 50：

```text
Potential connections = 40 × 50 = 2,000
```

如果数据库只适合处理 500 个连接，就会发生 Connection Storm。需要：

- 调整 Pool Size。
- 使用 RDS Proxy/PgBouncer。
- 控制 Auto Scaling 上限。
- 缩短 Transaction。
- 监控 Queue Wait Time。

## 十、增长率与未来容量

如果当前数据量为 10 TB，年增长 80%，三年后：

```text
10 × 1.8³ ≈ 58.3 TB
```

增长是复利，不是简单增加 `10 × 80% × 3`。

需要区分：

- 用户增长。
- 单用户行为增长。
- 数据保留时间变化。
- Feature 增加带来的 Payload 增长。
- Region 和 Tenant 扩展。

## 十一、完整估算示例

设计一个图片分享服务：

```text
DAU: 5 million
Uploads/user/day: 2
Average image: 3 MB
Reads/upload: 100
Peak factor: 5
Retention: permanent
Annual user growth: 50%
```

计算：

```text
Uploads/day = 10 million
Write QPS average ≈ 116
Write QPS peak ≈ 580
New storage/day ≈ 30 TB
New storage/year ≈ 10.95 PB before replication
Reads/day = 1 billion
Read QPS average ≈ 11,600
Read QPS peak ≈ 58,000
```

设计含义：

- 图片必须进入 Object Storage，而不是关系型数据库。
- 全球读取应由 CDN 承担。
- Metadata 与 Blob 分开存储。
- Upload 适合 Pre-signed URL，避免 Application Server 中转大文件。
- Lifecycle、Compression 和不同清晰度版本会显著影响成本。

## 十二、估算结果如何驱动架构

| 估算结果 | 可能的架构响应 |
| --- | --- |
| 读远大于写 | Cache、CDN、Read Replica |
| 大量 Blob | Object Storage、Pre-signed URL |
| 突发写入 | Queue、Buffer、Backpressure |
| 单表快速增长 | Partition、Archive、Retention |
| 网络成本高 | Compression、Locality、CDN |
| 长连接很多 | Event-driven Runtime、Connection Gateway |

估算不能直接证明必须使用某项技术，但可以帮助建立理由。

## 十三、常见错误

- 使用平均 QPS 忽略 Peak。
- 忘记 Replication、Index 和 Backup。
- 把 DAU 当作并发用户。
- 计算精确到小数点，却没有可靠输入。
- 只估算第一年，不考虑复合增长。
- 忽略 Egress Cost 和跨 Region Transfer。
- 不把估算结果映射回架构决策。

## 十四、自测题

1. 1 billion requests/day 大约是多少 Average QPS？
2. 20,000 RPS、平均延迟 300 ms 时并发请求约多少？
3. 为什么 Cache Hit Ratio 从 90% 提升到 99% 会显著降低后端压力？
4. 为什么数据库连接数不能随 Application Instance 线性无限增长？
5. 哪些存储开销经常在面试估算中被遗漏？

## 十五、总结

- 容量估算关注数量级、关键假设和架构影响。
- Provisioning 应基于 Peak、Headroom 和故障场景，而不是平均值。
- QPS、并发、存储、带宽和连接数需要一起考虑。
- 增长率应按复利计算，并考虑行为与 Feature 变化。
- 最重要的是解释数字为什么会改变架构选择。
