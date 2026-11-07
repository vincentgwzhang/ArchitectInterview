# 05. Schema Registry、事件版本与兼容性管理

## Event Contract

Event 是跨团队长期 Contract。包含 Event ID、Type、Schema Version、Occurred At、Aggregate ID 和 Payload。区分 Domain Event 与 Integration Event，后者应更稳定并隐藏内部模型。

## Schema Registry

集中存储 Avro/Protobuf/JSON Schema，自动检查 Compatibility，并支持 Producer/Consumer 解析。Registry 不替代语义治理：字段含义改变即使 Schema 兼容也可能破坏 Consumer。

## Compatibility

- Backward：新 Reader 读旧 Data。
- Forward：旧 Reader 读新 Data。
- Full：双向。

常见安全变更是增加带 Default 的 Optional Field。不要复用 Protobuf Field Number，不要随意收紧 Enum/Validation。

## 版本策略

优先在同一 Event Type 兼容演进。真正语义变化可发布新 Event Type/Version，并在迁移期双发。避免每次增加字段都创建 Topic v2。

## Consumer 治理

知道谁消费、使用哪些字段和当前 Lag。提供 Contract Test、Sample、Changelog、Deprecation 与 Sunset。Consumer 应容忍未知字段并避免依赖无承诺字段。

## Upcasting

读取旧 Event 时转换为当前内部表示，保留原始 Event 不修改。长 Retention/Replay 必须测试所有历史 Version。

## 总结

Schema Registry 自动保护结构兼容，语义兼容仍依赖 Ownership、文档、Consumer Visibility 和迁移纪律。
