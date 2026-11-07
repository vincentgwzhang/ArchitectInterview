# 02. Rolling、Blue-Green、Canary 与 Feature Flag

## Rolling

逐批替换实例，资源成本低。新旧版本共存，要求 API/Event/Schema 向后兼容；Rollback 重新滚动，速度取决于启动和批次。

## Blue-Green

两套完整环境，切换 Router。回滚快、验证充分，但基础设施成本高，Database 通常仍共享并限制真正回滚。

## Canary

先向少量用户/流量发布，比较 Error、Latency、SLO 和 Business Metric，再扩大。分流可按百分比、Tenant、Region 或 Cohort，需避免 Session 版本跳变。

## Feature Flag

代码部署与功能发布分离。支持内部用户、A/B、Kill Switch。Flag 有 Owner、Type、Expiry 和清理，不能替代长期权限系统。

## 选择

- Stateless 常规服务：Rolling。
- 高风险且有双环境预算：Blue-Green。
- 需要真实流量验证：Canary。
- 业务曝光控制：Feature Flag。

可组合：Rolling 部署 + Canary Traffic + Feature Flag Release。

## 自动验证

设置 Analysis Window、Baseline、Guardrail、最小样本和 Rollback。避免短时间低流量造成误判。

## 总结

部署策略控制技术风险，Feature Flag 控制业务曝光。数据库和外部 Side Effect 的兼容性仍需独立设计。
