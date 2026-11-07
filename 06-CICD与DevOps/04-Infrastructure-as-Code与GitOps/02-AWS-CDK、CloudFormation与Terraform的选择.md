# 02. AWS CDK、CloudFormation 与 Terraform 的选择

## CloudFormation

AWS Native 声明式服务，State 由 AWS Stack 管理，覆盖 AWS 新能力通常较快。局限是 AWS-only、模板复杂和部分错误反馈较慢。

## AWS CDK

使用 TypeScript/Java/Python 等定义 Construct，Synth 为 CloudFormation。适合工程团队复用高级 Construct 和测试。风险是 Imperative Language 可能产生过度抽象；部署语义仍是 CloudFormation。

## Terraform

多 Provider、生态广、语言相对简单，适合 Multi-cloud/SaaS 资源统一管理。团队需要安全管理 State 和 Provider Upgrade。

## 选择维度

- Cloud 范围。
- 现有技能与平台。
- State/Drift 模型。
- 新服务支持速度。
- Module/Construct 生态。
- Policy/Test。
- Import/Migration。
- Vendor Lock-in 与 TCO。

## 组合风险

可以 Terraform 管 Account/Foundation，CDK 管 Application Stack，但每个 Resource 必须只有一个 Owner，避免两个工具同时管理。

## 评估

用真实 Slice 做 PoC：VPC、EKS/Lambda、IAM、跨账户和 Rollback。比较 Plan 可读性、Pipeline、Drift、Test 和 Developer Experience。

## 总结

工具没有绝对优胜。AWS-only 团队可选 CDK/CloudFormation，多平台组织常选 Terraform；一致 Ownership 和成熟 Pipeline 比语法偏好更重要。
