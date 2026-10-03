# 本地工程基座：Oxlint 普通检查与 React/Hooks 能力补充

研究日期：2026-10-03（Asia/Shanghai）。状态：官方文档、固定发布元数据与固定源码核验；未安装、构建或运行验证。

本补充服务于[冻结本地工程基座的工具链与平台支持矩阵](https://github.com/WenshuaiDev/InfraNexus/issues/49)的最新选择：项目用 TypeScript 7 严格类型检查、Oxlint 基础及 React/Hooks 检查、Prettier 格式化；首期不引入 tsgolint，不引入 ESLint 或 TypeScript 6/5 兼容环境。本补充只核实 Oxlint 新增槽位，不重新推荐已移除的 OpenAPI 生成器或客户端。

## 精确稳定候选与平台

| 对象 | 精确值 / 核实事实 | 官方来源 |
| --- | --- | --- |
| Oxlint | **1.86.0**；npm `latest` 为该稳定版本；发布时间 2026-09-28T10:30:31.889Z | [固定元数据](https://registry.npmjs.org/oxlint/1.86.0)、[registry 发布记录](https://registry.npmjs.org/oxlint) |
| Node engines | `^20.19.0 || >=22.12.0`；已选 Node 24.21.0 满足该声明 | [固定元数据](https://registry.npmjs.org/oxlint/1.86.0) |
| Linux amd64 / glibc | `@oxlint/binding-linux-x64-gnu@1.86.0`；`os: linux`、`cpu: x64`、`libc: glibc` | [固定平台包元数据](https://registry.npmjs.org/%40oxlint%2Fbinding-linux-x64-gnu/1.86.0) |
| Linux arm64 / glibc | `@oxlint/binding-linux-arm64-gnu@1.86.0`；`os: linux`、`cpu: arm64`、`libc: glibc` | [固定平台包元数据](https://registry.npmjs.org/%40oxlint%2Fbinding-linux-arm64-gnu/1.86.0) |
| 固定源码身份 | tag `oxlint_v1.86.0` 指向 commit `2ae2939bb2fd98796393658b21556b2a2467e047` | [官方 tag](https://github.com/oxc-project/oxc/tree/oxlint_v1.86.0)、[固定源码](https://github.com/oxc-project/oxc/tree/2ae2939bb2fd98796393658b21556b2a2467e047) |

Oxlint 主包把各平台 binding 作为精确 `1.86.0` 的 optionalDependencies 分发；应在目标容器架构内安装并保留所需 optional dependency。存在官方平台包不等于已通过该平台安装、加载或 lint 测试。[主包元数据](https://registry.npmjs.org/oxlint/1.86.0)

## 普通 Oxlint 足以提供所选的基础与 Hooks 检查

Oxlint 的内置插件是原生规则组，并非必须安装的同名 ESLint npm 插件。`react` 组包含来自 React、React Hooks、React Refresh 的规则；本期所需 Hooks 规则无需额外插件版本，也无需安装 `eslint-plugin-react-hooks` 或 ESLint。`react` 默认未启用，必须显式加入插件列表。[内置插件说明](https://oxc.rs/docs/guide/usage/linter/plugins)

| 必须明确开启的规则 | 能力 | 分类 / 证据 |
| --- | --- | --- |
| `react/rules-of-hooks: error` | 检查 Hooks 的调用上下文、条件/循环调用与顺序问题 | `pedantic`；仅启用默认 correctness 不足。[官方规则页](https://oxc.rs/docs/guide/usage/linter/rules/react/rules-of-hooks)、[1.86.0 固定实现](https://github.com/oxc-project/oxc/blob/2ae2939bb2fd98796393658b21556b2a2467e047/crates/oxc_linter/src/rules/react/rules_of_hooks.rs) |
| `react/exhaustive-deps: error` | 检查 `useEffect` 等 Hooks 的依赖列表 | `correctness`；建议仍显式固定为 error，避免只记“打开 React 插件”。[官方规则页](https://oxc.rs/docs/guide/usage/linter/rules/react/exhaustive-deps)、[1.86.0 固定实现](https://github.com/oxc-project/oxc/blob/2ae2939bb2fd98796393658b21556b2a2467e047/crates/oxc_linter/src/rules/react/exhaustive_deps.rs) |

两个规则均由 Oxlint Rust 实现运行，并未标记为需要类型信息的规则。固定版本发布包的 schema 也包含它们。可以使用普通 Oxlint，无需 TypeScript 旧 Compiler API 或 tsgolint。[发布包](https://registry.npmjs.org/oxlint/-/oxlint-1.86.0.tgz)、[普通与类型感知检查职责](https://oxc.rs/docs/guide/usage/linter/type-aware)

基础检查与其他 React 检查应在实施时形成明确的有效规则清单；例如未使用变量、不可达代码、重复分支，以及 JSX key、重复 props 等使用普通规则即可。两项 Hooks 规则是本次明确要求的最低配置，不代表完整基础清单已在本研究中实现。自定义 effect hook 若需要依赖检查，应按 `react/exhaustive-deps` 的 `additionalHooks` 正则配置并用实际 hook 样例核验。[规则总表](https://oxc.rs/docs/guide/usage/linter/rules)、[依赖规则配置](https://oxc.rs/docs/guide/usage/linter/rules/react/exhaustive-deps)

## 配置边界

1. 使用 `plugins` 时写出全部所需内置组；该字段覆盖默认插件组，不能只写 `react` 却误认为默认的 TypeScript、Unicorn、Oxc 插件仍保留。`eslint` 在 Oxlint 中是内置基础规则组名称，不表示安装 ESLint 包。[插件与覆盖语义](https://oxc.rs/docs/guide/usage/linter/plugins)、[固定配置 schema](https://github.com/oxc-project/oxc/blob/2ae2939bb2fd98796393658b21556b2a2467e047/npm/oxlint/configuration_schema.json)
2. 明确启用 `react` 和上述两项 Hooks 规则；普通 lint 配置保持 `options.typeAware: false`、`options.typeCheck: false`，命令不加 `--type-aware` 或 `--type-check`。类型错误由锁定的 TypeScript 7.0.2 独立执行严格类型检查。[配置参考](https://oxc.rs/docs/guide/usage/linter/config-file-reference)、[类型感知模式](https://oxc.rs/docs/guide/usage/linter/type-aware)
3. 不把“React/Hooks 基础检查”扩大为 React 全规则。1.86.0 的 `react` 组还包含官方标注实验性的 React Compiler 检查；其中部分属于 correctness，例如 `react/error-boundaries`。因此整类开启前须审查最终规则清单，或使用显式清单、对实验规则逐项关闭，不能靠未开启 nursery 就断言没有实验 React Compiler 规则。`react/hooks` 与 `react/rules-of-hooks` 还存在覆盖重叠，不默认同时开启。[插件实验边界](https://oxc.rs/docs/guide/usage/linter/plugins)、[固定 error-boundaries 实现](https://github.com/oxc-project/oxc/blob/2ae2939bb2fd98796393658b21556b2a2467e047/crates/oxc_linter/src/rules/react/error_boundaries.rs)、[react/hooks 说明](https://oxc.rs/docs/guide/usage/linter/rules/react/hooks)
4. 本期不需要 JS 插件桥接、独立 ESLint 插件、React Compiler 或 tsgolint 依赖。Prettier 继续独立格式检查，Oxlint 配置不默认扩展到全部风格规则。这是按已确认能力范围给出的配置建议，而非已运行配置。

## 与 tsgolint 的能力分界

普通 Oxlint 负责源码解析、作用域/控制流等非类型感知检查；tsgolint 则基于 `typescript-go` 建立 TypeScript program，提供依赖类型信息的规则。Oxlint 1.86.0 的 `oxlint-tsgolint >=7.0.2003` peer 明确标为 **optional**，主包没有 TypeScript compiler 依赖，因此普通模式不要求安装它。[固定包元数据与 peerDependenciesMeta](https://registry.npmjs.org/oxlint/1.86.0)、[官方职责说明](https://oxc.rs/docs/guide/usage/linter/type-aware)

本期没有承诺 `typescript/no-floating-promises`、`typescript/no-unsafe-assignment` 等类型感知 lint，也不能把 `tsc --noEmit` 的严格类型检查等同于这些 lint 规则。将来如增加此类能力，需要重新讨论并锁定 tsgolint 及其类型系统兼容边界；本次不纳入其版本。[官方类型感知规则与启用要求](https://oxc.rs/docs/guide/usage/linter/type-aware)

## 尚未验证

- pnpm 12.8.1 在目标 linux/arm64 容器中的实际依赖解析、optional binding 安装与加载；linux/amd64 的实际执行。
- 固定 Oxlint 1.86.0 在项目 TS/TSX、React 19.3.0 与测试源码上的规则结果、误报和忽略范围。
- 生效配置确实包含两项 Hooks 检查；能拒绝条件 Hook、缺失 effect 依赖等反例，也能接受正常代码；自定义 Hooks 选项按实际源码验证。
- 最终有效规则列表不意外包含实验 React Compiler 或类型感知规则；普通启动不解析新版本。
- TypeScript 7 严格检查、普通 Oxlint、Prettier 三个独立检查在同一候选上的组合结果。

本研究只证明发布版本、声明范围、平台分发、固定实现与可配置能力；不构成工程安装、构建、测试或浏览器验收通过的证据。

## 保留请求校验且取消生成器的独立性核对

`nethttp-middleware v1.2.0` 的固定 go.mod 仅直接要求 `kin-openapi v0.142.0`，不要求 oapi-codegen 生成器或其 runtime。固定实现接受 `*openapi3.T` 并返回标准 `net/http` 中间件，调用 kin-openapi 的 `ValidateRequest`；因而可以保留它包装手写 handler。此为源码接口依据，尚未组合运行。响应校验仍需在契约测试中显式执行，不能认为请求中间件自动验证响应。[固定 go.mod](https://github.com/oapi-codegen/nethttp-middleware/blob/v1.2.0/go.mod)、[固定实现](https://github.com/oapi-codegen/nethttp-middleware/blob/v1.2.0/oapi_validate.go)
