# 02. EBS、EFS、FSx 与不同存储模型

## EBS

AZ 级 Block Storage，通常挂载单 EC2，适合 Boot、Database 和低延迟 Block Workload。gp3 通用，io2 面向高 IOPS/Durability。Snapshot 存 S3 管理层并可跨 Region/Account Copy。

实例跨 AZ 不能直接挂原 Volume，需要 Snapshot/Replication/重建。

## EFS

Regional NFS，多 AZ、多 Client 共享，弹性容量，适合共享内容、Home Directory、Lambda/ECS 文件。注意 Throughput Mode、Small-file Metadata、POSIX 权限和成本。

## FSx

托管特定文件系统：Windows File Server、Lustre、NetApp ONTAP、OpenZFS。适合 Windows Share、HPC、Enterprise NAS 等专业 Workload。

## 选择

- 单实例/数据库 Block：EBS。
- Linux 共享 NFS：EFS。
- Windows/SMB、HPC 或专业 NAS：FSx。
- 海量 Object/静态文件：S3。

## 可靠性

Storage Availability 不等于 Backup。定义 Snapshot/Backup、Retention、Cross-account Copy 和 Restore Test。加密 KMS、最小 Mount/Network 权限。

## 总结

Block、File、Object 解决不同 Access Semantics。选择应从 Protocol、Latency、共享、AZ、吞吐和恢复要求出发。
