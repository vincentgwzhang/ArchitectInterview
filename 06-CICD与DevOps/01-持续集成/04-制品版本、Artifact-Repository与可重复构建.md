# 04. 制品版本、Artifact Repository 与可重复构建

## Artifact

Jar、Container Image、Frontend Bundle、Helm Chart 和 IaC Module 都是可发布制品。Build Once，环境间 Promote 同一 Digest，不在 Staging/Prod 重新 Build。

## Version

使用 Semantic Version（Library/Public Product）或 Commit SHA/Build ID（Service）。Human-readable Tag 可变风险高，部署应固定 Immutable Digest。

## Repository

Artifact Repository 提供存储、访问控制、Retention、Promotion、Vulnerability Metadata 和审计。开发依赖与已批准生产制品可使用不同 Repository/Policy。

## Reproducible Build

- 锁定 Dependency/Toolchain/Base Image。
- 固定时区和 Locale。
- 移除时间戳/随机顺序等非确定输入。
- Hermetic Build 限制隐式网络和 Host 依赖。
- 保存 Build Metadata 与 SBOM。

完全 Bit-for-bit Reproducibility 较难，但至少确保相同 Source/Config 可追溯地产生等价 Artifact。

## Provenance 与签名

CI 使用受保护身份签名 Artifact，记录 Source、Builder、Workflow 和 Dependency。Deploy Policy 只允许可信 Registry、签名和通过 Scan 的制品。

## Retention

保留正在运行、可 Rollback、审计和长期支持版本；清理未引用 Snapshot。避免清理后无法回滚。

## 总结

Artifact 是交付链的不可变核心。版本、Digest、Repository、SBOM 和 Provenance 共同支持重现、晋级、回滚与审计。
