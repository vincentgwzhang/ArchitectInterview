# 04. AWS Backup、DR 演练与跨区域恢复

## AWS Backup

集中定义 Backup Plan、Vault、Retention、Cross-account/Region Copy 和 Compliance。用 Tag/Resource Selection 自动覆盖，但要检查新服务支持和实际成功率。

## 隔离

Backup Vault 放独立 Backup/Recovery Account，KMS 权限独立，Vault Lock 防恶意删除。生产管理员不应能同时删除全部 Backup。

## DR

选择 Backup/Restore、Pilot Light、Warm Standby 或 Active-active。IaC、AMI/Image、Config、Secret、DNS、Certificate 和 Dependency 都必须可恢复，不只是数据。

## 演练

定期在隔离 Account/Region Restore：

1. 恢复 Infrastructure 与 Data。
2. 验证权限/Key。
3. 运行 Data Integrity 与业务 Smoke Test。
4. 测实际 RTO/RPO。
5. 清理并记录 Gap。

## Failback

定义原 Region 恢复后如何同步新数据、Fencing、切回和验证。只测试 Failover 不完整。

## 总结

Backup 成功 Job 不等于可恢复。独立 Vault、跨区域 Copy、完整应用恢复和定期演练才能证明 DR。
