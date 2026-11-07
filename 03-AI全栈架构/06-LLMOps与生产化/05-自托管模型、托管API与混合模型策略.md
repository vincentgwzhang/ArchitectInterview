# 05. 自托管模型、托管 API 与混合模型策略

## 托管 API

优势：快速上线、无需 GPU 运维、模型能力更新快、弹性好。  
风险：Usage Cost、Rate Limit、Data Policy、Vendor Dependency、Model Behavior Change 和网络延迟。

## 自托管

优势：数据与部署控制、可定制、稳定版本、大规模时可能降低单位成本。  
代价：GPU Capacity、Serving、Quantization、Batching、Upgrade、Security、24×7 Operations 和专业人才。

开源 Weight 不等于免费生产服务。

## 评估维度

- Task Quality。
- Throughput/Latency/SLA。
- 峰值与平均 Utilization。
- Data Residency/Privacy。
- Model License。
- Hardware Availability。
- Engineering/On-call TCO。
- Fine-tuning 需求。
- Lock-in/Exit。

## 自托管 Serving

需要设计：

- GPU/Accelerator 类型。
- Tensor/Model Parallelism。
- Dynamic Batching。
- KV Cache 管理。
- Quantization。
- Autoscaling 与 Cold Start。
- Model Artifact Security。
- Canary 与 Rollback。

性能必须在真实 Prompt/Output Length 分布下测量。

## 混合策略

- 小 Model 处理分类/抽取。
- 托管强 Model 处理复杂推理。
- 敏感任务路由私有 Deployment。
- Provider A 为主，B 为经过验证的 DR。
- Offline Batch 使用自托管，实时使用 API。

Gateway 根据 Policy 和任务路由，并持续记录质量/成本。

## TCO

自托管 TCO 包括闲置 GPU、冗余、工程、升级和 Incident；托管 TCO 包括 Token、Egress、Premium Capacity 和 Vendor Support。使用不同增长场景做 Break-even 分析。

## Exit Strategy

保持 Eval Dataset、内部 Request/Response Contract、Data Export 和 Provider Adapter。完全可移植不现实，但应知道迁移成本。

## 总结

选择由质量、数据、规模、团队和 TCO 驱动。混合策略常能兼顾控制与能力，但也增加 Routing、Eval 和运营复杂度。
