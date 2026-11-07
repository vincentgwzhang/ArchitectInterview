# 02. Vite、Webpack、Babel、ESLint 与代码规范

## 工具职责

- Vite：开发服务器与 Build Tool，开发期利用 Native ESM，生产通常基于 Rollup。
- Webpack：高度可配置的 Module Bundler，适合复杂遗留生态与定制 Pipeline。
- Babel：JavaScript Syntax Transform 和 Plugin Ecosystem。
- TypeScript Compiler：类型检查，也可输出 JavaScript。
- ESLint：静态代码规则与错误检查。
- Prettier：确定性格式化。

不要把 Bundling、Transpilation、Type Checking 和 Lint 混成同一概念。

## Build Pipeline

```text
Source -> Transform -> Type Check/Lint -> Bundle -> Minify
       -> Asset Hash -> Source Map -> Artifact
```

Build 必须可重复，不应在 Build 时注入不可追溯的环境差异。推荐 Build Once、Promote Same Artifact，在 Runtime 注入允许变化的配置。

## Vite 与 Webpack 选择

Vite 通常提供更快开发反馈和较低配置；Webpack 在复杂 Loader、Module Federation 和成熟企业定制上仍有优势。选择依据应包括现有生态、Plugin、Migration Cost、Build Performance 和团队能力。

## ESLint

关键规则：

- React Hooks Dependency。
- TypeScript Unsafe Operation。
- Import Boundary 与循环依赖。
- Accessibility。
- Security-sensitive Pattern。
- 未使用代码。

Rule 应解释目的，并区分 Error 与 Warning。大量无人处理 Warning 会让 Signal 失效。

## Monorepo

共享 Base Config，但允许 Package 的合理差异。使用 Workspace、Task Cache 和受影响项目检测减少 CI 时间。锁定 Package Manager 与 Lockfile。

## 环境配置

发到 Browser 的变量都是公开信息，不能放 Secret。使用经过 Schema Validation 的 Public Config，并明确 Dev、Test、Staging、Prod 差异。

## Supply Chain

- 固定依赖版本与 Lockfile。
- Dependency Scan。
- 最小化安装 Script 权限。
- 生成 SBOM。
- 签名 Artifact。
- 定期升级而非多年积累。

## 总结

前端工具链应优化 Feedback、Reproducibility、Boundary 和 Security。工具越多不代表工程化越好，标准化和可维护性更重要。
