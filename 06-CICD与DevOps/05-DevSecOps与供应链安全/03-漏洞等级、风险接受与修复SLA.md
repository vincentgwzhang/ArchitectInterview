# 03. 漏洞等级、风险接受与修复 SLA

## 风险评估

不要只看 CVSS。结合：

- 是否可达/可利用。
- 是否暴露互联网。
- 是否有 Exploit。
- 数据/业务 Criticality。
- 现有 Mitigation。
- Fix 可用性。
- 攻击前置条件。

## 修复 SLA

示例应由组织风险确定：Critical 数小时/天，High 数天，Medium 数周。Internet-facing 和已利用漏洞使用更短 SLA。

## 修复方式

- Upgrade/Patch。
- Remove Dependency/Feature。
- Configuration Mitigation。
- Network/Permission Compensating Control。
- WAF 临时缓解。

验证 Fix 不只是版本变更，还要重新扫描和测试。

## Risk Acceptance

需要 Business/Technical Context、到期时间、Risk Owner、Compensating Control 和 Re-review。开发者不能永久自行接受组织级高风险。

## VEX

Vulnerability Exploitability eXchange 可表达某 CVE 对特定 Product 是否受影响及理由，减少无效噪音，但结论必须有证据并随版本更新。

## Metric

Open by Severity、Mean Age、SLA Breach、Recurrence、Exception Expiry、Critical Asset Coverage。不要用“发现数量”评价团队安全好坏。

## 总结

漏洞治理是持续风险管理。基于可利用性和资产影响排序，用 SLA、Owner、例外期限和验证形成闭环。
