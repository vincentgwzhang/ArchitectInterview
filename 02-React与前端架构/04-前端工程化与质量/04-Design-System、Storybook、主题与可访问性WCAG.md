# 04. Design System、Storybook、主题与可访问性（WCAG）

## Design System 的组成

Design System 不只是 Component Library，还包括：

- Design Principle。
- Token：Color、Spacing、Typography、Motion。
- Accessible Component。
- Pattern 与 Content Guideline。
- Documentation。
- Governance 和 Contribution Model。

## Token

使用语义 Token，而不是在产品代码写固定颜色：

```text
color.text.primary
color.surface.danger
space.component.md
```

语义层支持 Theme、Dark Mode、Brand 和可访问性调整。Token 需要版本化并跨 Design/Figma 与 Code 同步。

## Component API

优秀组件应：

- 默认 Accessible。
- 提供有限且清晰 Variant。
- 支持 Composition，而非巨大 Boolean Props。
- 保留 Escape Hatch，但不轻易破坏一致性。
- 明确 Controlled/Uncontrolled 行为。

## Storybook

用于隔离开发、文档、交互测试和 Visual Regression。Story 应覆盖 Default、Loading、Empty、Error、Long Content、Keyboard 和不同 Theme。

Storybook 不是生产集成测试的替代品，但可显著提高 Component Feedback。

## WCAG 与 Accessibility

核心原则 POUR：

- Perceivable。
- Operable。
- Understandable。
- Robust。

实践：

- 使用 Semantic HTML。
- 完整 Keyboard Navigation。
- 可见 Focus。
- 正确 Label 和 Error Association。
- 足够 Color Contrast。
- 不只靠颜色表达状态。
- Modal Focus Trap 与恢复。
- 尊重 Reduced Motion。

ARIA 不能修复错误交互模型。优先使用原生 `button`、`input` 和 `dialog` 能力。

## Governance

建立 Owner、Contribution RFC、Versioning、Deprecation 和 Migration Tool。用 Adoption、重复组件减少、Accessibility Defect 和交付速度衡量价值。

## 总结

Design System 是产品一致性、开发效率与 Accessibility 的共享平台。成功依赖 Token、可靠 Component、文档、治理和持续用户验证。
