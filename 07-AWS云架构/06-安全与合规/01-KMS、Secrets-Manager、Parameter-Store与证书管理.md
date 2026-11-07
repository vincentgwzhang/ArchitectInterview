# 01. KMS、Secrets Manager、Parameter Store 与证书管理

## KMS

管理加密 Key 和 Cryptographic Operation。Envelope Encryption 用 Data Key 加密数据，再用 KMS Key 加密 Data Key。Key Policy 是关键权限边界，结合 IAM、Grant、Rotation 和 CloudTrail。

不要把大量数据直接发送 KMS 加密；使用服务端加密或 Data Key。

## Secrets Manager

保存 Password/API Key，支持 Version、Rotation、Resource Policy 和跨账户。应用通过 Role 读取并短期 Cache。Rotation 需要新旧 Credential 过渡和验证。

## Parameter Store

保存配置与 SecureString，成本/功能较轻。复杂 Secret Rotation 与专门 Secret Lifecycle 更适合 Secrets Manager。

## Certificate

ACM 管理 Public/Private Certificate 并与 ALB/CloudFront/API Gateway 集成。监控不由 ACM 自动续期的证书、DNS Validation、Private CA 生命周期与客户端 Trust。

## 选择

- Encryption Key：KMS。
- 可轮换 Secret：Secrets Manager。
- 普通/层级配置：Parameter Store。
- TLS Certificate：ACM/Private CA。

## 总结

Key、Secret、Config 和 Certificate 是不同资产。使用短期身份、最小权限、自动轮换和审计管理完整生命周期。
