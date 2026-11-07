# 02. Build、Lint、Test、Scan 与质量门禁

## Pipeline 顺序

按反馈速度从快到慢：

```text
Format/Lint/Type -> Compile/Unit -> Component/Contract
-> Package -> Security Scan -> Integration/E2E -> Publish
```

可并行的 Stage 并行执行，失败快速取消无关工作。

## Build

锁定 Compiler/Dependency，使用干净环境，Build Once 并生成不可变 Artifact。记录 Commit、Dependency、SBOM 和 Provenance。

## 门禁

质量 Gate 应与风险关联：

- 编译/Lint Error：阻塞。
- Unit/Contract Failure：阻塞。
- Critical Vulnerability：按 Policy 阻塞。
- Coverage：关注关键变更与趋势，不迷信总百分比。
- 性能/Bundle：超过 Budget 阻塞或需要批准。

大量长期 Warning 会降低信号，应有 Owner 和清理策略。

## Flaky Test

自动 Retry 只用于收集证据，不应把 Flaky 变成绿色。Quarantine 有期限，跟踪 Flake Rate 和 Owner。

## Scan

SAST、SCA、Secret、Container、IaC。结果去重、按 Reachability/Exploitability 排序，并有修复 SLA。Scanner 故障时是否 Fail-open 由风险层级决定。

## Developer Experience

关键检查本地可复现；错误信息明确；使用 Cache 与 affected-only 缩短反馈。质量与速度不是对立，快速可靠 CI 能提高两者。

## 总结

CI 门禁保护可发布 Main。重点是快速、有行动意义、可复现，并让 Artifact 从一开始具备安全与追溯证据。
