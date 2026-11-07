# 02. SBOM、依赖签名、镜像签名与来源证明

## SBOM

Software Bill of Materials 列出 Component、Version、License 和关系，常见格式 SPDX/CycloneDX。它帮助快速回答“是否包含某漏洞依赖”，但不自动证明制品安全。

## Artifact Signing

对 Container、Package、Binary 进行签名，部署时验证签名、Issuer、Repository 和 Workflow Identity。Keyless Signing 可使用 OIDC 短期身份，减少长期私钥管理。

## Provenance

记录 Source Repository/Commit、Builder、Workflow、Dependency、Build Parameter 和 Artifact Digest，证明制品从何而来。SLSA 提供供应链成熟度思路。

## Dependency Trust

- Lockfile 与可信 Registry Proxy。
- Namespace/Package 防 Confusion。
- Review 新 Dependency。
- 限制 Install Script。
- Hash/Signature Verification。
- Renovation 与 Vulnerability Monitoring。

## Admission

Production Policy 只允许：受信 Registry、不可变 Digest、合法签名、合规 Provenance、无禁止漏洞/License 的 Artifact。

## 生命周期

每次 Build 生成 SBOM/Provenance，与 Artifact 一起存储。新 CVE 出现时重新扫描已部署 Digest，而不是等待重新 Build。

## 总结

SBOM 提供内容透明度，签名验证完整性和身份，Provenance 证明构建来源。三者结合才能建立可验证供应链。
