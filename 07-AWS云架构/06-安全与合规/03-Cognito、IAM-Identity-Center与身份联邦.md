# 03. Cognito、IAM Identity Center 与身份联邦

## Cognito

面向应用客户身份（CIAM），User Pool 处理注册、登录、MFA、Token；Identity Pool 可交换 AWS 临时 Credential。需要设计 Account Recovery、Custom Domain、Trigger、Tenant 和 Token Claim。

## IAM Identity Center

面向员工/Workforce 访问 AWS Account/Application，与企业 IdP Federation，使用 Permission Set 分发跨账户 Role。

## Federation

SAML/OIDC 让外部 IdP 负责身份生命周期，AWS/应用消费 Assertion/Token。验证 Issuer、Audience、Signature、Expiry、Nonce/State。

## 授权

认证后仍需应用 RBAC/ABAC 与 Resource Ownership。Group/Claim 映射有版本和大小限制，不能把所有业务权限永久编码在 Token。

## 生命周期

员工入离职通过 IdP/SCIM；客户 Consent/Deletion 通过产品流程。MFA、Break-glass、Session Duration 和 Access Review 按风险。

## 选择

Customer Login：Cognito/外部 CIAM。员工访问 AWS：Identity Center。应用企业 SSO：OIDC/SAML Federation。

## 总结

区分 Customer 与 Workforce Identity。Federation 减少长期 AWS User，但授权和账户生命周期仍需应用与组织治理。
