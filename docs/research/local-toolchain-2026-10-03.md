# 本地工程基座：工具链稳定版本与兼容约束研究

研究日期：2026-10-03（Asia/Shanghai）。状态：**事实研究与候选建议；版本尚未冻结，组合尚未构建。**

对应研究票：[核实本地工具链的稳定版本与兼容约束](https://github.com/WenshuaiDev/InfraNexus/issues/48)。所属地图：[本地工程基座：方案与工具链决策地图](https://github.com/WenshuaiDev/InfraNexus/issues/46)。范围依据：[确认纯本地工程基座的完整方案基线](https://github.com/WenshuaiDev/InfraNexus/issues/47#issuecomment-5968016780)。

## 结论与阅读顺序

本研究给出可确认的发布版本、工具约束、镜像身份和待验证项。**目前不能声称已有一套完全闭合且通过构建的工具链。** 主要未闭合项是 TypeScript 7 与原 OpenAPI 生成器的兼容路线；它属于后续版本冻结决策，不以降级项目 TypeScript 回避。

用户于本次研究中明确指出“typescript 5.X版本太低了”。因此 **TypeScript 5.x 不再作为项目编译器推荐或冻结候选**。本报告推荐以稳定 TypeScript 7.0.2 为目标，区分主编译器、供工具使用的兼容 API、生成器自身依赖；TypeScript 6 并存或独立生成环境均未获得用户确认。前端详细章节记录证据和备选路线。

现行[工具链冻结第一轮确认](https://github.com/WenshuaiDev/InfraNexus/issues/49#issuecomment-5968158135)已决定：macOS arm64 宿主机 + linux/arm64 容器为首轮完整验收必过环境；Linux 宿主机及 linux/amd64 为适配目标，未实测则标“待验证”，不阻塞首轮交付。版本需落在受维护的声明支持交集，普通启动不改版本，技术/主版本/支持边界变化需重新讨论。本研究沿用这些已确认原则，仅补齐具体版本、证据和兼容取舍，不重新提问。

| 范围 | 推荐目标候选 | 主要边界 |
| --- | --- | --- |
| Go / API | Go 1.26.8；chi 5.3.2；pgx 5.11.0；go-redis 9.22.0 | 直接 Go 下限有交集，实际模块闭包与组合编译未执行 |
| Go 工具 | sqlc 1.31.1；Goose 3.28.0；Air 1.67.4；golangci-lint 2.14.0 | 应用和工具独立锁；lint 官方二进制构建于 Go 1.27.0 |
| 契约 | OpenAPI 3.0.3；oapi-codegen 2.8.0；nethttp-middleware 1.2.0；kin-openapi 0.142.0；runtime 1.7.0 | 保守 dialect 候选；生成、运行时请求校验、响应测试各自需要证据 |
| Web | Node 24.21.0；pnpm 12.8.1；React 19.3.0；TypeScript 7.0.2；Vite 8.3.2；Tailwind CSS 4.3.3；shadcn CLI 4.21.1 | TS7 工具 API 与原生成器兼容路线待决，不能直接复制一组 latest 安装 |
| 数据 / 代理 | PostgreSQL 18.6；Redis 8.2.10；Nginx 1.30.5 | PG18 新卷布局；Nginx 动态 DNS 必须同时配置 |
| 浏览器 | @playwright/test 1.63.0 + 官方 v1.63.0-noble 镜像 | 包/镜像版本配对；runner 内 Node 精确 patch 尚未运行核验 |
| 宿主 | macOS Docker Desktop 4.93.0；Linux Engine/CLI 29.8.2 + Compose 5.5.1 | 候选支持边界；当前没有任何工程工作流验收 |

所有精确值的官方依据在下列对应章节。推荐并不等于各组件都是最高版本；本研究优先选择声明交集、当前支持系列和明确的版本身份。

## 研究方法与证据分级

- **官方事实**：官方文档、GitHub release/tag 源码、npm/Node/Go 发布元数据及官方 SBOM。
- **Registry 元数据**：只通过 HTTP 请求镜像 manifest/index，核对 amd64/arm64 descriptor，计算原响应 SHA-256 并与 registry digest 对比；未下载镜像层。
- **候选推断**：根据上述事实提出具体版本及兼容方案；尚未确认的选项需后续冻结票确认，已确认的平台与升级原则保持有效。
- **运行证据**：本次没有。没有安装依赖、拉取镜像、执行生成器、构建工程、启动 Docker 或运行测试。只新增研究文档，未实现工程代码或更改主机工具链。

来源可能继续更新。固定 tag / commit / digest 的证据用于指明本次选中的内容；可变官方文档和 registry dist-tag 只表示观测时的状态，不是永久事实。

## Go 后端与工具精确矩阵

建议 Go 基线保留 1.26 系列，使用 **Go 1.26.8**。官方下载 JSON 同时返回稳定 Go 1.27.1 和 1.26.8；选 1.26.8 是为了满足这些工具的共同最低要求并保留已支持系列，不代表它是最高版本。[下载 API](https://go.dev/dl/?mode=json)、[发行历史与支持政策](https://go.dev/doc/devel/release#go1.26.8)。

下表 Go 列是该组件精确 tag 的 `go.mod` 声明；工具的编译最低版本、生成代码最低版本和实际使用的 Go 工具链不是同一概念。日期来自对应 Release API 的 `published_at`（UTC 日期），Go 日期来自官方发行历史。

| 组件/模块 | 建议固定版本 | 发布日期 | tag 声明最低 Go | 官方来源 |
| --- | --- | --- | --- | --- |
| Go 工具链 | `1.26.8` | 2026-09-01 | — | [发行历史](https://go.dev/doc/devel/release#go1.26.8)、[下载 API](https://go.dev/dl/?mode=json) |
| chi `github.com/go-chi/chi/v5` | `v5.3.2` | 2026-08-20 | `1.23` | [Release](https://github.com/go-chi/chi/releases/tag/v5.3.2)、[go.mod](https://github.com/go-chi/chi/blob/v5.3.2/go.mod) |
| pgx `github.com/jackc/pgx/v5` | `v5.11.0` | 2026-09-07 | `1.25.0` | [Release](https://github.com/jackc/pgx/releases/tag/v5.11.0)、[go.mod](https://github.com/jackc/pgx/blob/v5.11.0/go.mod) |
| sqlc `github.com/sqlc-dev/sqlc` | `v1.31.1` | 2026-04-22 | `1.26.0`；建议工具链行 `go1.26.2` | [Release](https://github.com/sqlc-dev/sqlc/releases/tag/v1.31.1)、[go.mod](https://github.com/sqlc-dev/sqlc/blob/v1.31.1/go.mod) |
| Goose `github.com/pressly/goose/v3` | `v3.28.0` | 2026-09-02 | `1.26.0` | [Release](https://github.com/pressly/goose/releases/tag/v3.28.0)、[go.mod](https://github.com/pressly/goose/blob/v3.28.0/go.mod) |
| go-redis `github.com/redis/go-redis/v9` | `v9.22.0` | 2026-08-03 | `1.24` | [Release](https://github.com/redis/go-redis/releases/tag/v9.22.0)、[go.mod](https://github.com/redis/go-redis/blob/v9.22.0/go.mod) |
| Air `github.com/air-verse/air` | `v1.67.4` | 2026-08-01 | `1.26.0` | [Release](https://github.com/air-verse/air/releases/tag/v1.67.4)、[go.mod](https://github.com/air-verse/air/blob/v1.67.4/go.mod) |
| oapi-codegen `github.com/oapi-codegen/oapi-codegen/v2` | `v2.8.0` | 2026-07-17 | `1.25.0` | [Release](https://github.com/oapi-codegen/oapi-codegen/releases/tag/v2.8.0)、[go.mod](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/go.mod) |
| 请求校验 `github.com/oapi-codegen/nethttp-middleware` | `v1.2.0` | 2026-07-19 | `1.25.0` | [Release](https://github.com/oapi-codegen/nethttp-middleware/releases/tag/v1.2.0)、[go.mod](https://github.com/oapi-codegen/nethttp-middleware/blob/v1.2.0/go.mod) |
| OpenAPI parser/filter `github.com/getkin/kin-openapi` | **`v0.142.0`** | 2026-07-11 | `1.25` | [Release](https://github.com/getkin/kin-openapi/releases/tag/v0.142.0)、[go.mod](https://github.com/getkin/kin-openapi/blob/v0.142.0/go.mod) |
| 生成代码运行库 `github.com/oapi-codegen/runtime` | `v1.7.0` | 2026-08-16 | `1.24.0` | [Release](https://github.com/oapi-codegen/runtime/releases/tag/v1.7.0)、[go.mod](https://github.com/oapi-codegen/runtime/blob/v1.7.0/go.mod) |
| golangci-lint `github.com/golangci/golangci-lint/v2` | `v2.14.0` | 2026-09-24 | `1.26.0`（发布二进制实际 Go 见下文） | [Release](https://github.com/golangci/golangci-lint/releases/tag/v2.14.0)、[go.mod](https://github.com/golangci/golangci-lint/blob/v2.14.0/go.mod) |
| Staticcheck | 使用 golangci-lint 内置 `2026.2.1`，模块 `honnef.co/go/tools v0.8.1` | 2026-08-21 | `1.26.0` | [Release](https://github.com/dominikh/go-tools/releases/tag/2026.2.1)、[版本映射源码](https://github.com/dominikh/go-tools/blob/2026.2.1/lintcmd/version/version.go)、[go.mod](https://github.com/dominikh/go-tools/blob/2026.2.1/go.mod) |

`kin-openapi` 是刻意保守的选择：本次 latest endpoint 返回 `v0.149.0`（2026-08-28），但 `oapi-codegen v2.8.0` 与 `nethttp-middleware v1.2.0` 的各自 `go.mod` **都直接要求 `v0.142.0`**。上游明确警告 kin-openapi 处于 v0，升级可能发生破坏性变更；不应只因 latest 较新就独立升级。[最新 Release](https://github.com/getkin/kin-openapi/releases/tag/v0.149.0)、[codegen go.mod](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/go.mod)、[middleware go.mod](https://github.com/oapi-codegen/nethttp-middleware/blob/v1.2.0/go.mod)、[上游警告](https://github.com/oapi-codegen/nethttp-middleware/blob/v1.2.0/README.md#ive-just-updated-my-version-of-kin-openapi-and-now-i-cant-build-my-code-)。

## 兼容交集与事实边界

### Go 工具链与 lint

- 这些直接组件的 `go.mod` 声明共同最低值为 **Go 1.26.0**；选定的 Go 1.26.8 高于这些声明。sqlc 的 `toolchain go1.26.2` 是建议值，不是禁止 1.26.8 的上限。Go 官方规定 `go` 行是最低要求，主模块的 `go` 行必须不低于依赖的 `go` 行；`toolchain` 行是建议工具链，不能单独保证精确锁定。[Go toolchains](https://go.dev/doc/toolchain#module-and-workspace-configuration)。
- 如希望后续环境真正使用同一工具链，可在固定 Go 镜像里设置 `GOTOOLCHAIN=local`，配合版本预检，使依赖要求更高 Go 时直接失败；仅写 `toolchain go1.26.8` 且保留默认 `auto` 不构成精确锁定。此为根据官方选择规则形成的实施建议，未执行。[工具链选择](https://go.dev/doc/toolchain#go-toolchain-selection)。
- golangci-lint 的支持不是“当前主机装了新 Go 就可以”：官方要求其编译所用 Go 至少覆盖被分析代码的 Go，且新 Go 仍须各 linter 完成适配。[官方 FAQ](https://golangci-lint.run/docs/welcome/faq/#which-go-versions-are-supported)。
- `v2.14.0` 的精确 tag release workflow 指定 `GO_VERSION: '1.27.0'`；已进一步读取官方 **linux-amd64 和 linux-arm64 发布 SBOM**，两份的 `stdlib.versionInfo` 均为 `go1.27.0`。因此官方二进制的构建 Go 高于应用基线 1.26.8；这是发布元数据证据，未执行二进制。[release workflow](https://github.com/golangci/golangci-lint/blob/v2.14.0/.github/workflows/release.yml)、[amd64 SBOM](https://github.com/golangci/golangci-lint/releases/download/v2.14.0/golangci-lint-2.14.0-linux-amd64.tar.gz.sbom.json)、[arm64 SBOM](https://github.com/golangci/golangci-lint/releases/download/v2.14.0/golangci-lint-2.14.0-linux-arm64.tar.gz.sbom.json)。
- `golangci-lint v2.14.0` 的 go.mod 和 arm64 SBOM 都记录 `honnef.co/go/tools v0.8.1`，Staticcheck 官方源码把它映射为 `2026.2.1`。建议只启用 golangci-lint 中的 Staticcheck，而不引入第二套独立漂移的 Staticcheck 二进制。若确需独立工具，应固定相同模块版本。Staticcheck `2026.2` 已发布 Go 1.27 支持，`2026.2.1` 修复两类误报。[lint go.mod](https://github.com/golangci/golangci-lint/blob/v2.14.0/go.mod)、[版本映射](https://github.com/dominikh/go-tools/blob/2026.2.1/lintcmd/version/version.go)、[2026.2 Release](https://github.com/dominikh/go-tools/releases/tag/2026.2)、[2026.2.1 Release](https://github.com/dominikh/go-tools/releases/tag/2026.2.1)。

### OpenAPI 3.0 与 3.1

**建议当前共同基线固定 `openapi: 3.0.3`**；这不是说 3.1 完全不能用，而是当前研究足以支持的生成与运行时校验共同保守选择。

- 精确 `oapi-codegen v2.8.0` README 已明确支持 OpenAPI 3.0 和初始 3.1，包括 webhooks、`type: [T, "null"]`、`oneOf` + `const`。不能沿用旧版“oapi-codegen 不支持 3.1”的结论。它自己需 Go 1.25+，生成 Chi server 代码声明需 Go 1.24+。[tag README](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/README.md#does-oapi-codegen-support-openapi-31)、[支持的 server](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/README.md#supported-servers)。
- `nethttp-middleware` 是 kin-openapi `openapi3filter` 的包装，官方明确测试 Chi、gorilla/mux、net/http。生成 strict server 并不自动覆盖完整请求校验；应显式装配请求校验 middleware，并配置安全校验函数。[middleware README](https://github.com/oapi-codegen/nethttp-middleware/blob/v1.2.0/README.md)、[codegen 请求校验说明](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/README.md#requestresponse-validation-middleware)、[codegen 安全说明](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/README.md#implementing-security)。
- `kin-openapi v0.142.0` README 仍将 3.1 写为 `Soon`，但同 tag `openapi3/openapi3.go` 已有 3.1/3.2 版本识别、webhooks 和 JSON Schema dialect 字段。这是上游文档/实现成熟度信息不完全一致，**不能用读取 parser 字段的证据推导完整 3.1 运行时校验合规性**。本次未验证实际 3.1 请求/响应语义；留在 3.0.3 可避免把这个未核实范围带入当前基线。[README](https://github.com/getkin/kin-openapi/blob/v0.142.0/README.md)、[parser 源码](https://github.com/getkin/kin-openapi/blob/v0.142.0/openapi3/openapi3.go)。
- `oapi-codegen/runtime v1.7.0` 是额外必须固定的运行库，不能只锁生成器。此版修复参数绑定 panic，并加入受选项控制的 3.1 multi-type union 绑定；其 Release 说明新行为默认不会改变既有绑定。该版本超出生成器 tag 示例所用 `v1.5.0`，选择它是为了取绑定修复，仍须用 InfraNexus 的实际生成结果验证。[runtime Release](https://github.com/oapi-codegen/runtime/releases/tag/v1.7.0)、[生成器示例 go.mod](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/examples/go.mod)。
- 运行时请求校验不可代替响应契约测试。oapi-codegen 文档仍注明 middleware 未提供 HTTP 响应校验；kin-openapi 本身有独立的 `ValidateResponse` API，可用于测试或专门装配，但不能说接入上述 middleware 后响应已自动校验。[codegen 请求/响应校验说明](https://github.com/oapi-codegen/oapi-codegen/blob/v2.8.0/README.md#requestresponse-validation-middleware)、[kin-openapi 验证示例](https://github.com/getkin/kin-openapi/blob/v0.142.0/README.md#validating-http-requestsresponses)。

### 数据库、Redis 与生成器

- pgx v5.11.0 的文档给出 PostgreSQL >=14，列出的上游测试矩阵为 PostgreSQL 14–18；因此 PostgreSQL 18 系列在其声明范围内。文档中的“Go 1.25+”是该 tag 下限；本项目 1.26.8 在范围内。[pgx tag README](https://github.com/jackc/pgx/blob/v5.11.0/README.md#supported-go-and-postgresql-versions)。
- sqlc v1.31.1 明确支持 `sql_package: pgx/v5`；schema 可以指向 Goose migrations 目录，sqlc 识别 Up/Down 并忽略 Down，但 sqlc 自己不执行迁移。文件按字典序读取，建议统一固定宽度序号或固定宽度时间戳，确保顺序一致。[pgx/v5 guide](https://docs.sqlc.dev/en/v1.31.1/guides/using-go-and-pgx.html)、[Goose migration parsing](https://docs.sqlc.dev/en/v1.31.1/howto/ddl.html#goose)。
- sqlc v1.31.1 自身依赖 pgx v5.9.2，Goose v3.28.0 依赖 pgx v5.10.0；应用选 v5.11.0 与工具内部依赖不要求强行完全相同。若工具和应用共用一个 Go module，MVS 会影响选择结果，不能仅凭这张表声称已经得到实际锁定图。Goose postgres driver 精确源码导入 `pgx/v5/stdlib`。[sqlc go.mod](https://github.com/sqlc-dev/sqlc/blob/v1.31.1/go.mod)、[Goose go.mod](https://github.com/pressly/goose/blob/v3.28.0/go.mod)、[Goose postgres driver](https://github.com/pressly/goose/blob/v3.28.0/cmd/goose/driver_postgres.go)。
- go-redis v9.22.0 README 列出支持 Redis 8.0、8.2、8.4、8.8、8.10；Redis 7+ 只是非正式的 should-work 说明。若容器矩阵选择 8.2 系列，则处于该明确支持列表。连接、超时、TTL 及后续实际使用的命令仍需真实集成测试。[go-redis 支持声明](https://github.com/redis/go-redis/blob/v9.22.0/README.md#supported-versions)。
- Air 的用途是 Go 开发期 live reload。该 tag README 的容器示例仍写 Go >=1.25，但 **go.mod 已要求 1.26.0，应以机器可执行声明为下限**。容器文件监听如需轮询，示例配置提供 `poll` / `poll_interval`；是否启用应由目标 Docker 文件同步行为决定，未实测。[Air README](https://github.com/air-verse/air/blob/v1.67.4/README.md)、[go.mod](https://github.com/air-verse/air/blob/v1.67.4/go.mod)、[配置例](https://github.com/air-verse/air/blob/v1.67.4/air_example.toml)。


## 前端工具链与 TypeScript 7 兼容边界

用户已明确拒绝将 TypeScript 5.x 作为项目版本；主工程候选改为 **TypeScript 7.0.2**。其 npm 稳定版与官方发布公告均已核实。但原有 `openapi-typescript 7.13.0`、`typescript-eslint 8.71.0` 与一个统一的 `typescript@7.0.2` 包之间，**不存在本次能够证明的全稳定兼容交集**。这是待冻结决策，而不是通过关闭 peer 检查可以消除的问题。

- `openapi-typescript 7.13.0` 的 peer 是 `typescript: ^5.x`，发布包实际导入旧 TypeScript 编译器 API；其 `next` 标签还是 `7.0.0-rc.1`，同样要求 `^5.x`，不能当作支持 6/7 的更新。[稳定元数据](https://registry.npmjs.org/openapi-typescript/7.13.0)、[标签元数据](https://registry.npmjs.org/openapi-typescript)、[发布源码包](https://registry.npmjs.org/openapi-typescript/-/openapi-typescript-7.13.0.tgz)
- `typescript-eslint 8.71.0` 支持 `>=4.8.4 <6.1.0`；`canary 8.71.1-alpha.7` 仍是同一范围，且为预发布。不能声称它们已经原生支持 TypeScript 7。[稳定元数据](https://registry.npmjs.org/typescript-eslint/8.71.0)、[canary 元数据](https://registry.npmjs.org/typescript-eslint/8.71.1-alpha.7)、[官方支持范围](https://typescript-eslint.io/users/dependency-versions/)
- 官方给出 TypeScript 7 检查器与 TypeScript 6 API 并存的 alias 路线，可作为**有官方依据、待组合验证**的候选；它可处理 lint 的旧 API 需求，仍不能满足 `openapi-typescript` 的 `^5.x` peer。[官方 TypeScript 7 公告](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)
- ESLint 9.x 于 2026-08-06 EOL，因此采用仍在维护的 ESLint 10.12.0 候选。`eslint-plugin-react 7.37.5`、`eslint-plugin-jsx-a11y 6.10.2` 的 peer 不包括 ESLint 10；基座原决策没有强制这两个插件，本报告不把它们加入推荐集合，也不为了它们退回 ESLint 9。[ESLint 支持周期](https://eslint.org/version-support/)、[react 插件元数据](https://registry.npmjs.org/eslint-plugin-react/7.37.5)、[a11y 插件元数据](https://registry.npmjs.org/eslint-plugin-jsx-a11y/6.10.2)

### 前端精确稳定候选

每个版本号链接指向本次实际 GET 的 npm 固定版本元数据；`—` 表示包未声明该项，而非没有任何间接要求。npm scoped 包链接采用 URL 编码。

#### 运行时、构建、UI、契约

| 包/工具 | 精确候选 | engines / peers / 状态 |
|---|---|---|
| Node.js | **24.21.0** | 官方 dist index 的 24.x 当前条目；2026-09-07，Krypton LTS，npm 11.19.0；Linux x64/arm64 可用。备用 22.x 候选为 **22.23.3**（2026-09-23）。[dist index](https://nodejs.org/dist/index.json) |
| pnpm | [12.8.1](https://registry.npmjs.org/pnpm/12.8.1) | 包 engines `node >=18.*`；pnpm 12 为原生可执行程序，官方 npm 安装器另需 Node >=22.13；见后文安装边界。 |
| typescript | [7.0.2](https://registry.npmjs.org/typescript/7.0.2) | Node >=16.20.0；2026-07-08 发布；原生 `tsc`；主工程候选，旧 API 工具兼容待决。 |
| react | [19.3.0](https://registry.npmjs.org/react/19.3.0) | Node >=0.10.0；2026-09-09 发布。 |
| react-dom | [19.3.0](https://registry.npmjs.org/react-dom/19.3.0) | peer `react ^19.3.0`；与 react 精确同版。 |
| @types/react | [19.3.0](https://registry.npmjs.org/%40types%2Freact/19.3.0) | 元数据 `typeScriptVersion 5.6` 是最低类型语法支持信息，不是项目编译器冻结为 5.6。 |
| @types/react-dom | [19.3.0](https://registry.npmjs.org/%40types%2Freact-dom/19.3.0) | peer `@types/react ^19.3.0`；`typeScriptVersion 5.6`。 |
| @types/node | [24.19.1](https://registry.npmjs.org/%40types%2Fnode/24.19.1) | 24.x 类型线；`typeScriptVersion 5.6`；无需与 Node patch 数字相同。 |
| vite | [8.3.2](https://registry.npmjs.org/vite/8.3.2) | Node `^20.19.0 || >=22.12.0`；依赖 rolldown `~1.2.11`；2026-10-01 发布。 |
| @vitejs/plugin-react | [6.1.1](https://registry.npmjs.org/%40vitejs%2Fplugin-react/6.1.1) | Node `^20.19.0 || >=22.12.0`；必须 peer `vite ^8.0.0`；react-compiler/babel/oxc-transform-react peers 可选，不默认增加。 |
| tailwindcss | [4.3.3](https://registry.npmjs.org/tailwindcss/4.3.3) | 包未声明 engines；浏览器硬边界见下。 |
| @tailwindcss/vite | [4.3.3](https://registry.npmjs.org/%40tailwindcss%2Fvite/4.3.3) | peer `vite ^5.2.0 || ^6 || ^7 || ^8`；精确依赖 tailwindcss、node、oxide 4.3.3；oxide Node >=20。 |
| shadcn CLI | [4.21.1](https://registry.npmjs.org/shadcn/4.21.1) | Node >=20.18.1；组件是生成并拥有的源码，不把 CLI 版号当所有 UI 源码版本。 |
| openapi-typescript | [7.13.0](https://registry.npmjs.org/openapi-typescript/7.13.0) | **现有选择的兼容阻断，未纳入 TS7 可用主集合**：peer `typescript ^5.x`。 |
| openapi-fetch | [0.17.0](https://registry.npmjs.org/openapi-fetch/0.17.0) | 无 TS peer/编译器运行时依赖；依赖 openapi-typescript-helpers `^0.1.0`；消费 `paths` 类型。此事实不证明任何替代生成器输出可直接接入。 |

Node 24 的维护阶段按官方计划将于 2026-10-20 开始、EOL 为 2028-04-30；Node 22 已在 Maintenance、EOL 2027-04-30。这里选择 24 为新工程候选；不是用 Node 26 Current 替代 LTS。[官方 Release schedule](https://raw.githubusercontent.com/nodejs/Release/main/schedule.json)

#### 代码检查与格式化

| 包 | 精确候选 | engines / peers / 状态 |
|---|---|---|
| eslint | [10.12.0](https://registry.npmjs.org/eslint/10.12.0) | Node `^20.19.0 || ^22.13.0 || >=24`；使用 flat config。 |
| @eslint/js | [10.0.1](https://registry.npmjs.org/%40eslint%2Fjs/10.0.1) | 同 Node 范围；peer eslint `^10.0.0`（optional）；不要求与 eslint patch 相同。 |
| typescript-eslint | [8.71.0](https://registry.npmjs.org/typescript-eslint/8.71.0) | Node `^18.18.0 || ^20.9.0 || >=21.1.0`；peer eslint `^8.57.0 || ^9.0.0 || ^10.0.0`，TS `>=4.8.4 <6.1.0`；**需 TS6 API 方案或其他 lint 决策，不能与单一 TS7 包直接冻结**。 |
| eslint-plugin-react-hooks | [7.1.1](https://registry.npmjs.org/eslint-plugin-react-hooks/7.1.1) | Node >=18；peer eslint 包括 `^10.0.0`。 |
| eslint-plugin-react-refresh | [0.5.7](https://registry.npmjs.org/eslint-plugin-react-refresh/0.5.7) | peer eslint `^9 || ^10`。 |
| eslint-config-prettier | [10.1.8](https://registry.npmjs.org/eslint-config-prettier/10.1.8) | peer eslint >=7；关闭冲突格式规则，不代替 Prettier 执行。 |
| prettier | [3.9.9](https://registry.npmjs.org/prettier/3.9.9) | Node >=14；稳定版；不选 next 4.0.0-alpha.13。 |

仅选 ESLint 核心配 `@eslint/js` 并不能解析 TS/TSX，也不能代替 TS lint：如不接受 TS6 API 兼容层，ESLint 只能覆盖 JS 配置等范围，必须显式说明 TS/TSX lint 能力缺口，另行决定 parser/规则路线，不能把“ESLint10可安装”写成“TS7 lint已覆盖”。ESLint10 自身已删除 eslintrc 等旧入口。[ESLint10 发布说明](https://eslint.org/blog/2026/02/eslint-v10.0.0-released/)

#### 单元与浏览器测试

| 包 | 精确候选 | engines / peers / 状态 |
|---|---|---|
| vitest | [5.0.3](https://registry.npmjs.org/vitest/5.0.3) | Node `^22.12.0 || ^24.0.0 || >=26.0.0`；必须 vite `^6.4.0 || ^7.0.0 || ^8.0.0`；可选 jsdom `*`、@types/node `^22.0.0 || >=24.0.0`。 |
| @vitest/coverage-v8 | [5.0.3](https://registry.npmjs.org/%40vitest%2Fcoverage-v8/5.0.3) | 需要 coverage 时启用，与 vitest **精确同版**；不单独漂移。 |
| jsdom | [30.1.1](https://registry.npmjs.org/jsdom/30.1.1) | Node **`^22.22.2 || ^24.15.0 || >=26.0.0`**；可选 canvas `^3.2.3`，当前不默认增加。 |
| @testing-library/react | [16.3.3](https://registry.npmjs.org/%40testing-library%2Freact/16.3.3) | Node >=18；react/react-dom `^18.0.0 || ^19.0.0`；@types 同范围且 optional；**必须 @testing-library/dom ^10.0.0**。 |
| @testing-library/dom | [10.4.2](https://registry.npmjs.org/%40testing-library%2Fdom/10.4.2) | Node >=18；作为显式开发依赖，避免遗漏 RTL peer。 |
| @testing-library/user-event | [14.6.7](https://registry.npmjs.org/%40testing-library%2Fuser-event/14.6.7) | Node >=12；peer @testing-library/dom >=7.21.4。 |
| @testing-library/jest-dom | [7.0.1](https://registry.npmjs.org/%40testing-library%2Fjest-dom/7.0.1) | Node >=22；DOM peer >=10 <11；Vitest peer >=0.32（optional）。 |
| @playwright/test | [1.63.0](https://registry.npmjs.org/%40playwright%2Ftest/1.63.0) | Node >=20；精确依赖 playwright 1.63.0；与镜像、浏览器包同版。 |

React 19.3.0 落在 RTL 的明确支持范围内；无需基于 React19 这一点回退 React18。Vitest 的 DOM matcher 入口使用 `@testing-library/jest-dom/vitest` 并在 setupFiles 加载；不因为包名带 jest 就额外安装 Jest。[官方 matcher 用法](https://github.com/testing-library/jest-dom#with-vitest)

### Node、构建平台与浏览器边界

对已列出的运行时、构建和测试包，Node22/24 两条 LTS 的最严格显式 engines 门槛来自 jsdom30.1.1：22.x 至少22.22.2，24.x至少24.15.0。候选24.21.0和备用22.23.3满足这些元数据约束。只写“Vite需22.12+”会漏掉 jsdom；Node20不满足该测试集合。[jsdom30.1.1固定源码](https://raw.githubusercontent.com/jsdom/jsdom/v30.1.1/package.json)、[Vitest5.0.3固定源码](https://raw.githubusercontent.com/vitest-dev/vitest/v5.0.3/packages/vitest/package.json)、[Vite8.3.2固定源码](https://raw.githubusercontent.com/vitejs/vite/v8.3.2/packages/vite/package.json)

pnpm12 原生可执行程序与 Node 的角色需要分开：registry包声明 Node>=18，而官方推荐的 `get-pnpm` 安装器（本次 latest 0.0.5）需 Node>=22.13。候选Node24覆盖两者；Linux glibc x64/arm64均在官方分发支持表内。本期不执行该安装器或安装脚本。[pnpm安装说明](https://pnpm.io/installation)、[get-pnpm0.0.5元数据](https://registry.npmjs.org/get-pnpm/0.0.5)

TypeScript7、pnpm12、Rolldown 和 Tailwind oxide 带平台原生包；应在目标 Linux 容器内按架构安装，不复制 macOS 宿主 node_modules，不统一禁用 optionalDependencies。后续应让 lockfile 包含目标平台需要的记录，执行 frozen install 后核对实际二进制。这是从包的 optionalDependencies/os/cpu 推导的工程约束，本次没有安装验证。[TS7元数据](https://registry.npmjs.org/typescript/7.0.2)、[Rolldown1.2.11元数据](https://registry.npmjs.org/rolldown/1.2.11)、[Tailwind oxide4.3.3元数据](https://registry.npmjs.org/%40tailwindcss%2Foxide/4.3.3)

Tailwind4核心最低范围是 Chrome111、Safari16.4、Firefox128；某些新utility仍有更高单独要求。降低Vite JS target或加JS polyfill不能推导为弥补Tailwind所需CSS能力。若后续产品浏览器承诺低于该范围，需要重开样式技术或浏览器支持决策。[Tailwind兼容文档](https://tailwindcss.com/docs/compatibility)

### TypeScript7 与旧工具API：候选路线必须透明

TypeScript7.0原生编译器不提供兼容旧版的稳定 Compiler API；当前包导出 `. -> lib/version.cjs`，其余 API 路径带 `unstable`。公告对7.1 API是预期，不能写成已发布或已兼容。官方提供同时安装7检查器和6 API的方法；本次查到兼容包实际版本为 `@typescript/typescript6 6.0.2`，内部 `@typescript/old` 依赖为 `npm:typescript@^6`，当前稳定6.x是6.0.3。其发布源码 `lib/typescript.js` 是 `module.exports = require("@typescript/old")`。[TS7发布公告](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)、[TS7固定元数据](https://registry.npmjs.org/typescript/7.0.2)、[兼容包元数据](https://registry.npmjs.org/%40typescript%2Ftypescript6/6.0.2)、[TS6.0.3元数据](https://registry.npmjs.org/typescript/6.0.3)、[兼容包源码包](https://registry.npmjs.org/@typescript/typescript6/-/typescript6-6.0.2.tgz)

待冻结讨论的精确alias候选（不是本次修改）：

```json
{
  "devDependencies": {
    "@typescript/native": "npm:typescript@7.0.2",
    "typescript": "npm:@typescript/typescript6@6.0.2"
  }
}
```

官方设计由7的包提供 `tsc`、6兼容包提供 `tsc6`；让旧工具从 `typescript` 导入6 API。兼容包内部的 `^6` 必须由lockfile锁住本次已核实6.0.3，不能误说6.0.2兼容包意味着实际API patch一定6.0.2。typescript-eslint声明范围涵盖6.0.3，因此这一路线有静态依据；pnpm别名解析、工具实际入口、React TSX、lint规则和TS7检查仍须在后续同一候选上验证。

**这不是把主工程改为TS6，也不代表用户已同意双API工具链。** 直接把项目编译器改为6.0.3是另一条可讨论路线，用户没有选择；而且仍然不能满足openapi-typescript的 ^5.x peer，所以单独降为6也没有解决原生成器冲突。

前述TS7官方说明推导的配置风险应在实施时显式处理：新版默认 `types: []`、rootDir规则与废弃选项有变化；按应用、Node工具和测试分开写明确的types/rootDir，不依赖旧模板隐式默认。这里不替未来的组合测试作通过结论。

### OpenAPI生成器槽位：当前没有可证明的无缝替代

已核查候选及限定结论：

| 候选 | 本次查到的稳定版本/事实 | 与当前openapi-fetch paths的关系 |
|---|---|---|
| 保留 openapi-typescript | 7.13.0；实际tarball中多个transform模块 import ts from typescript，peer ^5.x。 | 原生输出paths；若保留只能讨论独立、明确锁版的旧API生成工具环境，与TS7主工程隔离；这会保留工具内部TS5，**待用户取舍，不能暗中采用**。 |
| @hey-api/openapi-ts | [0.99.0](https://registry.npmjs.org/%40hey-api%2Fopenapi-ts/0.99.0)；Node>=22.18，peer `>=5.5.3 || >=6.0.0 || 6.0.1-rc`；发布tarball仍直接import ts from typescript。 | 宽peer在数值上涵盖7，不足以证明旧API能接7；官方输出Data/Responses/definitions，Fetch client是它自己的客户端，未找到官方保证输出可直接替换openapi-fetch的paths。改它属于生成器/客户端决策，不是版本更新。 |
| Orval | [8.39.0](https://registry.npmjs.org/orval/8.39.0)；Node>=22.18；主包无TS peer，core8.39.0未声明TS依赖。 | 只能说明值得进一步调查；未证明其生成输出具有openapi-fetch所需paths结构，不作无缝替换承诺。 |
| swagger-typescript-api | [13.13.0](https://registry.npmjs.org/swagger-typescript-api/13.13.0)；Node>=20，直接依赖TS ^6.0.3。 | 可见工具内部自带TS6路线；未证明生成openapi-fetch paths；仍是另一种生成器/客户端选择。 |
| 预发布 | TS next7.1.0-dev.20261003.1；typescript-eslint canary8.71.1-alpha.7；Hey API next0.0.0-next-20260930190945。 | 不作为稳定候选；未来API/无TS依赖变化不能倒推当前稳定版已支持。 |

Hey API官方类型输出是每endpoint的请求Data和Responses，client-fetch是自家生成客户端；这与openapi-fetch从paths按URL/method进行类型推导的形状不同。本次未发现官方稳定的“只换生成器、原样保留openapi-fetch paths”的保证；若要保留openapi-fetch而改生成器，需另外证明paths的parameters/requestBody/responses/content结构、可选性和错误状态类型等，而不是只比较有没有TypeScript类型输出。[Hey API类型输出](https://heyapi.dev/docs/openapi/typescript/plugins/typescript)、[Hey API Fetch客户端](https://heyapi.dev/docs/openapi/typescript/clients/fetch)、[openapi-fetch官方用法](https://openapi-ts.dev/openapi-fetch/)

冻结票需要决断的具体问题：接受“主工程TS7 + lint用TS6 API + 生成器独立旧API环境”吗；若不接受工具内部TS5，是否改生成器及必要时客户端；若只要单一typescript7包，TS lint和契约生成的原组合当前无法冻结。不能静默降低主工程版本、强制peer、采用预发布，或本轮自行造适配生成器。

### shadcn 版本锁法

CLI4.21.1固定版本、components.json、组件源码、组件依赖分别锁定：固定CLI不能阻止远端registry内容变化。每次生成记录CLI版号、base/style/preset、registry地址、源commit（可用时）和内容摘要，把最终生成源码和精确直接依赖、pnpm-lock.yaml一同提交；组件更新作为可审查的源码diff，不在每次install/dev时重新下载覆盖。官方支持读取本地路径/URL；GitHub registry支持完整40字符commit SHA，且默认分支不是固定版本。上述是由其机制推导的可复现建议。[CLI文档](https://ui.shadcn.com/docs/cli)、[GitHub registry refs](https://ui.shadcn.com/docs/registry/github)

官方明确组件支持Tailwind4和React19，但当前CLI支持不同base（base/radix/aria），还需要沿用已确认选择或在冻结时明确选择，不能凭“shadcn”自动补一个未确认primitive库。具体组件依赖随实际引入的组件确定，本报告不空加整套库。[Tailwind4/React19说明](https://ui.shadcn.com/docs/tailwind-v4)

### Playwright 镜像、浏览器与架构

推荐候选包 `@playwright/test=1.63.0` 与 `mcr.microsoft.com/playwright:v1.63.0-noble` 同版；镜像有浏览器和系统依赖，**不含项目的Playwright npm包**，仍需项目安装同版依赖。官方指出包/镜像不匹配会找不到浏览器；Firefox/WebKit基于glibc，不用Alpine/musl替代。[官方Docker文档](https://playwright.dev/docs/docker)

固定v1.63.0的browsers.json显示：Chromium/Headless Shell153.0.8010.12、revision1243；Firefox155.0、revision1543；WebKit26.6、revision2359（部分mac14另有revision override）。这些是Playwright绑定的浏览器集合，不等于用户机器Chrome/Safari版本。[固定浏览器清单](https://raw.githubusercontent.com/microsoft/playwright/v1.63.0/packages/playwright-core/browsers.json)

固定版本官方Noble Dockerfile `NODE_VERSION=24`，通过NodeSource安装major24，并非精确patch；不能从这份Dockerfile推断镜像内一定是24.21.0。后续要么执行核实镜像Node patch并记录为测试运行时，要么在明确需求后调整镜像；本次未运行容器。[固定Dockerfile](https://raw.githubusercontent.com/microsoft/playwright/v1.63.0/utils/docker/Dockerfile.noble)

镜像部分已通过 Registry HTTP manifest 核验（未 pull/run）：

| 镜像 | Manifest list / index digest | 架构 |
|---|---|---|
| node:24.21.0-bookworm-slim | sha256:0e0ff40c39bc087845bfb27465a0df4ea419520094bc35842ff83dd8cbe6f9b6 | linux/amd64、linux/arm64 |
| mcr.microsoft.com/playwright:v1.63.0-noble | sha256:eff16c30e6f3f4af0a03fa4b706120d5e9b0891c344a27d64559aff5900a4a27 | linux/amd64、linux/arm64 |

完整 HTTP 查询方法见下方“官方镜像精确矩阵”。建议固定tag+index digest，保留架构选择；manifest双架构不等于两个架构的测试都已通过。

### 前端研究边界

可确认：所列版本存在且为稳定版本、包声明的engines/peer范围、已指出的冲突、固定发布源码的API依赖，以及官方运行模型。不可确认：完整依赖图可解析、安装脚本成功、TS7+API6+测试组合实际通过、UI组件交互/浏览器验收、实际镜像内工具patch、任何生产就绪结论。下一步是冻结兼容取舍及后续定向实现验证；不是把本报告中的候选当作一套已经验收的工程基座。

## 官方镜像精确矩阵

| 角色 | 推荐精确 tag | 选择理由及边界 |
|---|---|---|
| PostgreSQL | `docker.io/library/postgres:18.6-bookworm` | 官方当前受支持 18 系列补丁；全新库可直接使用 18 的卷布局。17.11 也受支持，但无需仅为旧路径选择 17。 |
| Redis | `docker.io/library/redis:8.2.10-bookworm` | 8.2 属于 Extended 系列，官方表列 EOL 2030-09-01；8.10.2 是当前较新的 Standard，而非本次稳定基线的必选项。 |
| Nginx | `docker.io/library/nginx:1.30.5-trixie` | 官方当前 stable 系列已是 1.30，1.30.5 发布于 2026-09-15；不应再将 1.28 写成当前 stable。 |
| Go 工具镜像 | `docker.io/library/golang:1.26.8-bookworm` | 与 Go 子研究确定的 1.26.8 对齐；显式 Debian 发行版避免浮动默认发行版。 |
| Node 工具镜像 | `docker.io/library/node:24.21.0-bookworm-slim` | 与前端子研究确认的 Node 24 LTS 对齐；此处选最小工具基础镜像，缺少的工具必须显式写入后续 Dockerfile。 |
| Playwright 浏览器运行器 | `mcr.microsoft.com/playwright:v1.63.0-noble` | 官方 Ubuntu 24.04 LTS 镜像，配合精确 `@playwright/test@1.63.0`；镜像已含浏览器及系统依赖，项目包仍需安装。 |

这些是工程选择建议，不是“任意两组件必然兼容”的实测结论。PostgreSQL 18 的官方支持期至 2030-11-14；17 至 2029-11-08。Redis 的支持期、Docker 发行版安全维护期、基础镜像更新频率属于不同维度，不能将数据库 EOL 当作底层 Debian 镜像的安全更新承诺。

来源：[PostgreSQL 版本政策](https://www.postgresql.org/support/versioning/)、[Redis Open Source 版本管理](https://redis.io/docs/latest/operate/oss_and_stack/install/version-mgmt/)、[Nginx 1.30 变更日志](https://nginx.org/en/CHANGES-1.30)、[Go 官方镜像清单](https://github.com/docker-library/official-images/blob/master/library/golang)、[Node 官方镜像清单](https://github.com/docker-library/official-images/blob/master/library/node)、[Playwright Docker 文档](https://playwright.dev/docs/docker)。

### Registry 实际核验：推荐镜像

以下 digest 均为本次实际 GET 返回的 **顶层 OCI image index digest**，不是某一个架构的 image manifest digest。每条 `Docker-Content-Digest` 均与对 HTTP 响应原始字节计算的 SHA-256 完全相同。固定顶层 index 能在同一引用下保留 amd64/arm64 的原生选择；不能把单架构 digest 当成多架构 index。

| 精确 tag | 顶层 index digest | 本次平台证据 |
|---|---|---|
| `postgres:18.6-bookworm` | `sha256:3725f4e2499eef5134592b3b4ab79a543ed7f8e533b05b5b637af926630f6650` | linux/amd64, linux/arm64/v8 |
| `redis:8.2.10-bookworm` | `sha256:164c759a0c342ee69d08fc99219382b0fd682181465c0df2e0e6911f4c85d73c` | linux/amd64, linux/arm64/v8 |
| `nginx:1.30.5-trixie` | `sha256:b972f831f200b19ef0767938224f9711e74cd783718738cd7405d5cabf75c442` | linux/amd64, linux/arm64/v8 |
| `golang:1.26.8-bookworm` | `sha256:a688600ca24f8a4d3ca77f95b0dd40704a9fc787c826660eb7ba0b641b8b175d` | linux/amd64, linux/arm64/v8 |
| `node:24.21.0-bookworm-slim` | `sha256:0e0ff40c39bc087845bfb27465a0df4ea419520094bc35842ff83dd8cbe6f9b6` | linux/amd64, linux/arm64/v8 |
| `mcr.microsoft.com/playwright:v1.63.0-noble` | `sha256:eff16c30e6f3f4af0a03fa4b706120d5e9b0891c344a27d64559aff5900a4a27` | linux/amd64, linux/arm64 |

每个推荐镜像所需的两种平台均存在，不需要把 Compose 全局硬锁为 `linux/amd64`。这里只证明 registry 发布了相应平台的 manifest，尚不能声称已在 Intel/AMD 或 Apple Silicon 主机成功运行。

#### 每种平台的 image manifest digest

- `postgres:18.6-bookworm`：[本次查询 endpoint](https://registry-1.docker.io/v2/library/postgres/manifests/18.6-bookworm)
  - `linux/amd64`：`sha256:9e73daeb439141c2b11eea2463f5f1a3b269fd90d897b41cddb7cb440f21aa5d`
  - `linux/arm64`：`sha256:4c6516b5d6dfd96a6888541396f76545a63290be0aec2542547d7bcdd7515e28`
- `redis:8.2.10-bookworm`：[本次查询 endpoint](https://registry-1.docker.io/v2/library/redis/manifests/8.2.10-bookworm)
  - `linux/amd64`：`sha256:a2f2c0b14d1e30e66599825fa299282cf55cd16edc8624961cf7d87d7fa4d7f4`
  - `linux/arm64`：`sha256:b77eddbcd5045575abc83811f5664b8ff024067e75dc34dcdd3904f99731ec12`
- `nginx:1.30.5-trixie`：[本次查询 endpoint](https://registry-1.docker.io/v2/library/nginx/manifests/1.30.5-trixie)
  - `linux/amd64`：`sha256:3d2f995522ddb52c3a4eb8008b8fa26f184f7146851df1da831f88885ac40aae`
  - `linux/arm64`：`sha256:444d474369737d45215e4c3019f87e3f657b6a9c09e6ec1e9b97a5f161223bac`
- `golang:1.26.8-bookworm`：[本次查询 endpoint](https://registry-1.docker.io/v2/library/golang/manifests/1.26.8-bookworm)
  - `linux/amd64`：`sha256:abe4f87f354c4f6d7ee3fb11b241c6b6c24a32ca50a2ebcc30493a2168e14048`
  - `linux/arm64`：`sha256:37a6d96e606ca410603609f2c235f182ff13060266d7626688b21f132ae338d4`
- `node:24.21.0-bookworm-slim`：[本次查询 endpoint](https://registry-1.docker.io/v2/library/node/manifests/24.21.0-bookworm-slim)
  - `linux/amd64`：`sha256:5cbc7caba8c2c0f0bca675d1b61b9f2857e1cf1853c6164ee9dd409501a936e7`
  - `linux/arm64`：`sha256:24b8bc17702002d2ed0c1da9ad66c3ee507cc279d0856726662ff2b6fc35c149`
- `mcr.microsoft.com/playwright:v1.63.0-noble`：[本次查询 endpoint](https://mcr.microsoft.com/v2/playwright/manifests/v1.63.0-noble)
  - `linux/amd64`：`sha256:bc6ab0d6d44ff4826e4cb8c1e6d801e185bfc42bb0753f8e2a30efc70db054c7`
  - `linux/arm64`：`sha256:a0f4498920a5dbac63196d9140ed738ef00470f27e2e74029abd8850b7bd5717`

Docker Hub endpoint 直接浏览可能返回 401；按下文向公开 auth endpoint 取得仅限 pull 范围的短期 bearer token 后访问。MCR 此公开仓库 endpoint 本次无需 token。

### PostgreSQL 18 卷布局的确定事实

官方 18.6 Bookworm Dockerfile 设置 `PGDATA=/var/lib/postgresql/18/docker`，声明 `VOLUME /var/lib/postgresql`。因此新 Compose 的命名卷应挂到 `/var/lib/postgresql`，而不是机械沿用 `/var/lib/postgresql/data`。17.11 Dockerfile 的默认 `PGDATA` 和 `VOLUME` 则仍为 `/var/lib/postgresql/data`。

18+ 的父目录挂载使未来跨大版本升级可采用 `pg_upgrade --link` 的目录布局，但**更换镜像 tag 或更换挂载路径本身不完成数据迁移**；官方 PostgreSQL 版本政策仍要求大版本升级通过 dump/reload 或 pg_upgrade，并阅读升级说明。对当前全新基线，直接创建正确布局；以后升级已有库应独立安排备份、升级与回退。

来源：[18.6 固定提交 Dockerfile](https://github.com/docker-library/postgres/blob/e00e1bd34ec5c8a8e7ad89b273b3d42efaf6d5bc/18/bookworm/Dockerfile)、[17.11 固定提交 Dockerfile](https://github.com/docker-library/postgres/blob/2603e26e245e558218728ee14e0a42dcb020dc7f/17/bookworm/Dockerfile)、[官方 PGDATA 说明](https://github.com/docker-library/docs/blob/master/postgres/README.md#pgdata)、[PostgreSQL 版本政策](https://www.postgresql.org/support/versioning/)。

### Nginx 动态 DNS：版本与配置一起成立

`upstream` 内 `server <domain> resolve` 从开源 Nginx 1.27.3 起可用；此前属于商业功能。它会监测域名对应 IP 的变化而无需重启 Nginx。必要配置是：upstream 使用共享内存 `zone`，并在 `http` 或该 `upstream` 中声明 `resolver`。仅升级镜像、仅写服务名，或仅加 `resolver` 都不等于完成这条配置。

Compose 自定义网络使用 Docker 内嵌 DNS；官方说明地址为 `127.0.0.11`。以下仅为后续实现时可采用的配置形状（服务名、端口、zone 大小、TTL 覆盖值需结合最终拓扑确定，尚未运行 `nginx -t`）：

```nginx
upstream api_backend {
    zone api_backend 64k;
    resolver 127.0.0.11 valid=5s;
    server api:8080 resolve;
}
```

`valid=5s` 是演示用的 TTL 覆盖值，不是官方要求；省略时遵循 DNS 响应 TTL。动态 DNS 功能应在实现阶段通过后端容器替换/IP 改变后代理恢复的针对性验收确认。1.30.5 满足功能版本下限，也属于本次观测的 stable 系列。

来源：[Nginx resolve/zone/resolver 指令](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server)、[Nginx 1.27.3 变更记录，收录于 1.30 日志](https://nginx.org/en/CHANGES-1.30)、[Docker 网络 DNS](https://docs.docker.com/engine/network/#dns-services)。

### Debian、Alpine、glibc 的边界

Redis 官方说明：Alpine 变体以 musl 为 C 库，适合镜像体积优先的场景；依赖 glibc 行为的附加软件可能遇到兼容性问题。8.2.10 Bookworm Dockerfile 确实基于 `debian:bookworm-slim`。建议此基线使用 Debian，理由是减少工具和本地库的变体，而不是声称 Redis Alpine 本身不受支持。8.2.10 的 Debian 和 Alpine Dockerfile 都对 amd64、arm64 启用 modules 构建；不能写成“Alpine 无模块”。Redis volume 是 `/data`。

来源：[Redis 官方镜像说明](https://github.com/docker-library/docs/blob/master/redis/README.md)、[Redis 8.2.10 Debian Dockerfile](https://github.com/redis/docker-library-redis/blob/78ec6fbd0dcf6ea97b4828244ae5f7ec4d04d01a/debian/Dockerfile)、[同版本 Alpine Dockerfile](https://github.com/redis/docker-library-redis/blob/78ec6fbd0dcf6ea97b4828244ae5f7ec4d04d01a/alpine/Dockerfile)。

Node 官方镜像说明：Bookworm 是 Debian 12，Trixie 是 Debian 13；slim 只带运行 Node 所需的最小软件，官方一般推荐常规镜像作为通用基础。这里选择 slim 的前提是后续将 Git、证书或编译工具等实际所需依赖写明，不能假定 slim 自带完整构建工具。Node 文档也区分 Alpine 的 musl 构建和 Debian 的 glibc 构建。

来源：[Node 官方镜像变体说明](https://github.com/nodejs/docker-node#image-variants)。

Playwright 有更严格的独立要求：Firefox/WebKit 的官方浏览器构建依赖 glibc，官方明确不支持 Alpine/musl；因此 Playwright runner 选官方 Noble。文档要求项目 Playwright 版本与镜像版本匹配，否则可能找不到浏览器。

来源：[Playwright Docker 镜像与 Alpine 限制](https://playwright.dev/docs/docker#image-tags)。

#### Playwright 内置 Node 的精确度不能混写

`v1.63.0` 的 `Dockerfile.noble` 写 `ARG NODE_VERSION=24`，从 NodeSource `node_24.x` apt 仓库安装 Node，**源码没有将内置 Node 固定为 24.21.0**。本次 MCR digest 固定了完整已发布镜像内容，但仅从这一 Dockerfile 和 manifest，不能得出 runner 中 `node --version` 的精确 patch。后续实际启动 runner 时应记录该 patch；若要求所有 Node 容器必须恰好相同 patch，需要额外设计派生 runner 镜像并验证浏览器兼容，不宜在研究结论里假装已经成立。

来源：[Playwright v1.63.0 固定版本 Dockerfile.noble](https://github.com/microsoft/playwright/blob/v1.63.0/utils/docker/Dockerfile.noble)。

### 备选镜像的 registry 观测

这些 tag 同样实际 GET 成功，并含 linux/amd64 和 linux/arm64；不是本轮默认选择。

| 备选 tag | 顶层 index digest | 用途 |
|---|---|---|
| `postgres:17.11-bookworm` | `sha256:639ab7ceb90e13123085b741fb31ef493fba25463002f6da665352e7b534b652` | 只有明确要保留 PostgreSQL 17 大版本时选择 |
| `golang:1.26.8-trixie` | `sha256:eae2aaa6add2936cbf350dd0d2628b363461542f0c4b3c0b558957e0f2997379` | 若后续统一迁到 Debian 13 可评估 |
| `node:24.21.0-trixie-slim` | `sha256:8ec5d7557396cfe32d21c3f9c13072355ceab22b584578ca4bb28af31120cffe` | 若后续统一迁到 Debian 13 可评估 |

### 镜像核验方法与证据边界

1. 读取 Docker Official Images 的 [postgres](https://github.com/docker-library/official-images/blob/master/library/postgres)、[redis](https://github.com/docker-library/official-images/blob/master/library/redis)、[nginx](https://github.com/docker-library/official-images/blob/master/library/nginx)、[golang](https://github.com/docker-library/official-images/blob/master/library/golang)、[node](https://github.com/docker-library/official-images/blob/master/library/node) 元数据，核对精确 tag、声明架构与构建 GitCommit。对 Playwright 使用官方 Docker 文档与固定版本 Dockerfile。
2. Docker Hub：GET `https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/<repo>:pull`，仅在内存中使用返回的 bearer token；再 GET `https://registry-1.docker.io/v2/library/<repo>/manifests/<tag>`。MCR：GET `https://mcr.microsoft.com/v2/playwright/manifests/v1.63.0-noble`。
3. Accept 头声明 OCI index、Docker manifest list、OCI image manifest、Docker image manifest 四种类型；本次全部返回 `application/vnd.oci.image.index.v1+json`。
4. 保存响应 `Date`、`Docker-Content-Digest`，对响应原始字节计算 SHA-256，比对一致；读取 `manifests[].platform` 和对应 digest。`unknown/unknown` 描述符没有被算作运行架构。
5. 本报告表格保存了本次观测的顶层及平台 digest。Registry 响应 Date 为 2026-10-03 10:16:19–22 UTC，即北京时间 18:16:19–22。短期认证 token 不写入报告。

本次证据等级：官方文档事实 + registry 元数据。确认的是上述精确 tag 在观测时存在，index 有双架构 descriptor，digest 可对应本次响应内容。未检查/下载镜像 layers；未做漏洞扫描；未证明任一主机能连接 registry、成功 pull、启动容器、安装前后端依赖、连接数据库、运行浏览器或通过验收。实施前若刷新 tag，应重新查询并记录 tag+index digest；tag 本身可重指向，digest 是内容固定引用。

## 宿主机、Compose 与执行平台候选

本节区分已确认的平台验收原则与尚待冻结的具体宿主工具版本。研究只执行了只读元数据请求和 `uname`、`git --version`、`make --version`；未调用 Docker。当前宿主机观测为 Darwin/arm64、Git 2.55.0、GNU Make 3.81，不能据此认定 Docker 或工程工作流已通过。

| 层 | 推荐候选 | 已核实依据与边界 |
| --- | --- | --- |
| macOS 容器宿主 | Docker Desktop 4.93.0；内置 Engine 29.8.1，Compose 以 5.5.1 为候选 | 官方发布说明记录 4.93.0 于 2026-09-28 发布并更新 Engine；4.91.0 引入 Compose 5.5.1，后续条目没有列出 Compose 更新。后者是由增量发布记录推断，必须在实施时读取实际版本，不能当作安装包检查结论。[Desktop releases](https://docs.docker.com/desktop/release-notes/#4930) |
| Linux 容器宿主 | Ubuntu 24.04 LTS + Docker Engine/CLI 29.8.2 + Compose plugin 5.5.1；Buildx 0.37.1 为构建候选 | Ubuntu 官方安装支持列表包括 24.04、amd64、arm64；Engine release 确认 29.8.2；Compose 5.5.1 发布于 2026-09-03；Buildx 0.37.1 列在 Desktop 4.92.0 更新。该组合未运行。[Ubuntu](https://docs.docker.com/engine/install/ubuntu/)、[Engine release](https://github.com/moby/moby/releases/tag/docker-v29.8.2)、[Compose release](https://github.com/docker/compose/releases/tag/v5.5.1)、[Buildx release](https://github.com/docker/buildx/releases/tag/v0.37.1) |
| 项目支持下限建议 | Compose 5.5.1；实际支持集合由冻结票确认，不承诺所有未来 major | 这是减少旧版 SDK/API 差异的支持策略建议，不是 Compose 文件语法的理论下限。5.6.0 已于 2026-10-02 发布，本研究选用较早的 5.5.1 作为候选，不需要最新 release 的 jobs 等能力。[5.6.0 release](https://github.com/docker/compose/releases/tag/v5.6.0) |
| Make / shell | Makefile 保持 GNU Make 3.81 可用的薄入口；主机脚本优先 POSIX sh | 工程设计建议，依据当前 Mac 已有版本，避免无必要要求宿主机额外安装新版 Make/Bash。能否做到需实现时在实际宿主验证；不新增 Python/Node/Go 为主机必需项。 |

macOS 应处于所选 Docker Desktop 支持的系统范围。Docker 官方政策为当前及前两个 macOS 大版本，并分别提供 Apple Silicon 与 Intel 安装包；不能把这一厂商范围等同于 InfraNexus 全部验收通过。[Docker Desktop for Mac](https://docs.docker.com/desktop/setup/install/mac-install/)

| 候选宿主 / 容器目标 | 定位与确认状态 | 需要的证据 |
| --- | --- | --- |
| macOS arm64 → linux/arm64 | 已确认首轮完整验收必过环境 | 真实本地 Docker 工作流、bind mount 热更新、入口和浏览器链路 |
| Linux amd64 → linux/amd64 | 已确认适配目标；未验证不阻塞首轮交付 | 当前只有官方包/镜像元数据；没有原生 Linux 宿主实测，不声称已通过 |
| macOS amd64 → linux/amd64；Linux arm64 → linux/arm64 | 元数据可行候选，是否列正式支持由冻结票决定 | 相应宿主上的真实验证；镜像存在并不覆盖宿主文件权限、监听和文件事件行为 |
| arm64 宿主 → linux/amd64 仿真 | 可作为后续补充实验，不作为默认运行方式 | 仿真结果不能替代原生 amd64 或原生 Linux 宿主验收 |

按已确认原则，本地设备不足时保留“待验证平台”，不阻塞首轮交付，也不新增必须采购设备或接入 CI 的要求。Linux 容器跑在 Mac 的 VM 中，不是 Linux 原生宿主验收。[平台与验收范围确认](https://github.com/WenshuaiDev/InfraNexus/issues/49#issuecomment-5968158135)

### 最小 Compose 特性集合

以下是已确认基线需要的能力，实施不得无意增加更高版本特性。`version: "3.8"` 等文件字段不构成 CLI 版本锁。

| 能力 | 用途及限制 | 官方依据 |
| --- | --- | --- |
| 明确 project name、独立 network/named volumes、`-f` 多文件覆盖、环境变量替换、profiles / 一次性 `run --rm` | 区分开发和每次测试；迁移/生成/测试按需运行。禁止固定共享 `container_name`；普通停止保留卷 | [Project name](https://docs.docker.com/compose/how-tos/project-name/)、[Merge](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/)、[Profiles](https://docs.docker.com/compose/how-tos/profiles/)、[Run](https://docs.docker.com/reference/cli/docker/compose/run/) |
| `depends_on.condition: service_healthy`、`healthcheck` | 等待依赖初次就绪。进程已启动不等于健康；运行中断连仍由应用处理 | [Startup order](https://docs.docker.com/compose/how-tos/startup-order/) |
| `up --wait --wait-timeout` | 有限时间内等候 running/healthy；仍需独立的整个环境检查 | 这些 flags 已直接核实存在于 [v2.20.2 up.go](https://github.com/docker/compose/blob/v2.20.2/cmd/compose/up.go)；[CLI reference](https://docs.docker.com/reference/cli/docker/compose/up/) |
| `pull_policy: never` 或显式 `--pull never`、`--no-build` | 普通启动只使用已准备的镜像，缺失时明确提示运行初始化；联网准备与更新显式执行 | [Pull policy](https://docs.docker.com/reference/compose-file/services/#pull_policy)、[Up flags](https://docs.docker.com/reference/cli/docker/compose/up/) |
| `127.0.0.1:入口端口:容器端口`，内部服务不发布端口 | 保持已确认回环入口；显式调试覆盖只增加回环端口 | [Port publishing](https://docs.docker.com/engine/network/port-publishing/) |

可选属性不是默认要求：`depends_on.restart` 自 Compose 2.17.0 起，`required` 以官方 release 2.20.2 为可确认边界，`healthcheck.start_interval` 文档标明 2.20.2；后者还依赖 Engine API 的支持。本基线不需要 Compose Watch、include、生命周期 hooks、provider 或 jobs。Air/Vite 处理源码热更新。`service_completed_successfully` 可以描述显式初始化的串行任务，但不能让普通启动偷偷执行数据库迁移。[Services reference](https://docs.docker.com/reference/compose-file/services/)、[2.20.2 release](https://github.com/docker/compose/releases/tag/v2.20.2)

因此 **2.20.2 只能作为上述语法/flags 的已核实较早版本，不是本项目的完整最低兼容承诺**。Compose v2.20.2 自带 Docker 24 SDK，不能由“语法存在”推断它可以连接所有 Engine 29；Engine 29.0–29.2 的 API 下限是 1.44，而当前 29.3–29.8 表列为 1.40。推荐 5.5.1 与当期 Engine 组合，实际 `doctor` 应检查 client/server/Compose/Buildx 和 API 协商。[v2.20.2 go.mod](https://github.com/docker/compose/blob/v2.20.2/go.mod)、[v5.5.1 go.mod](https://github.com/docker/compose/blob/v5.5.1/go.mod)、[Engine API matrix](https://docs.docker.com/reference/api/engine/)

回环绑定也是版本相关行为：Docker 文档注明 Engine 28.0.0 之前，同二层网络上的其他主机可能访问 localhost 发布端口。因此不建议用旧 Engine 作为本基线支持候选。[Port publishing warning](https://docs.docker.com/engine/network/port-publishing/)

## 锁定与更新建议

版本标签加 digest、精确依赖与锁文件、普通启动不改变版本、独立升级和按影响范围验证等原则已获第一轮确认；以下补充具体实施候选，不表示本轮已经创建了依赖文件或镜像。[锁定与升级确认](https://github.com/WenshuaiDev/InfraNexus/issues/49#issuecomment-5968158135)

1. **Go 应用与工具分开锁。** 应用提交 `go.mod`、`go.sum`；为生成器、迁移、sqlc、Air 记录精确版本及工具安装来源。`go` 指令是最低版本，`toolchain` 是建议版本，都不单独保证实际编译器精确锁定。固定 Go 基础镜像 digest，并在正式容器命令使用 `GOTOOLCHAIN=local`，避免执行时自动下载新工具链；不通过 `go env -w` 修改用户环境。官方预编译工具的版本、架构、校验和独立记录，尤其区分 golangci-lint 自身构建用 Go 与它检查的项目 Go。[Go toolchains](https://go.dev/doc/toolchain)、[Go modules reference](https://go.dev/ref/mod)
2. **前端锁实际解析结果。** 顶层直接依赖使用精确版本，提交 `pnpm-lock.yaml`，容器内 `pnpm install --frozen-lockfile`；同时固定 pnpm、Node 镜像、TypeScript 编译命令和工具兼容包。编译器、ESLint API 提供者及 OpenAPI 生成器不是同一兼容层。别名包装包若还有版本范围依赖，也必须通过 lockfile 记录实际内部包版本。[pnpm install](https://pnpm.io/cli/install)、[TypeScript 7 官方并存方案](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)
3. **镜像用精确 tag 加 index digest。** 保存人可读的版本/发行版标签，同时将实际引用固定到本报告所列 `sha256`。amd64 与 arm64 共享顶层 index，但验收记录还应保存实际选中平台 manifest。派生工具镜像新增 OS 包时须记录包版本/来源并固定最终构建镜像身份；基础镜像 digest 不会自动锁住后续从活动 apt 源安装的包。不能由基础镜像和 lockfile 就声称整个构建已经可复现。[Docker pin base image versions](https://docs.docker.com/build/building/best-practices/#pin-base-image-versions)
4. **生成结果单独保存。** OpenAPI 契约、生成器配置和生成文件共同提交；shadcn 的已生成源码、配置和组件来源也入库。固定 CLI 版本不等于冻结远端组件 registry。生成差异检查应在临时目录产生输出，不修改工作区后再宣称无漂移；这是已确认基线要求。
5. **升级显式进行。** 普通启动不重新解析版本或安装依赖；初始化/更新命令准备版本固定的工具镜像和缓存。升级时一次审查相关兼容链：Go 与 lint；Node 与 pnpm/Vite/Vitest/jsdom；TypeScript 与生成器/lint；Playwright 包与浏览器镜像；数据库 major 与迁移/备份恢复。先保留基线，再以新证据更新；研究日期不是永久版本承诺。

## 留给实施阶段的验证

下列均未执行。本研究关闭不等于这些检查通过，也不授权本轮执行。

| 验证对象 | 要解决的实际不确定性 |
| --- | --- |
| 首次依赖解析与工具镜像构建 | Go 模块选择后的实际图、pnpm peer/alias 解析、native 可选包、所选平台的二进制、生成器可运行性；不能用忽略 peer 约束的参数制造成功 |
| TypeScript 7 链路 | `tsc` 确实来自锁定的 7.x；ESLint 使用已明确的兼容 API；OpenAPI 生成文件可供 7.x、openapi-fetch、React 消费；显式核对 Vite/测试配置类型与生成漂移 |
| Go 与契约 | oapi-codegen、runtime、请求校验中间件、kin-openapi、chi 的实际组合编译；合法/非法请求、统一错误对象、真实 PostgreSQL/Redis 交互 |
| 本地源码与依赖卷 | Mac 文件挂载、Go 自动重启、Vite HMR、缓存隔离、Linux UID/GID 与权限；镜像架构存在不覆盖这些行为 |
| 服务生命周期 | PG18 挂载路径和数据保留、schema 不兼容失败、显式迁移、PG/Redis 中断与恢复、Nginx DNS 随 API/Web 替换恢复 |
| 浏览器运行器 | Playwright 包/镜像/浏览器配对，runner 内 Node patch 和系统依赖；经 Nginx 的页面、API、HMR、键盘/焦点/缩放链路 |
| 隔离与交付证据 | 测试独立项目与数据卷，清理不影响开发/其他工作区，导出后恢复至隔离环境；正式工程基座验收绑定同一候选 SHA、实际命令和环境 |

实施期日常验证继续按影响范围；最终工程基座验收使用同一稳定候选。当前没有工程代码，因此本次只检查研究文档与来源的一致性，不运行全仓验收。

## 后续冻结票需要作出的选择

由[冻结本地工程基座的工具链与平台支持矩阵](https://github.com/WenshuaiDev/InfraNexus/issues/49)承接，仍需与用户实时确认：

- 项目 TypeScript 5.x 已由用户明确排除。是否采用 TypeScript 7 主编译器及官方 TypeScript 6 API 并存方案；原 OpenAPI 生成器的 ^5.x peer 冲突如何处理，允许独立工具环境还是调整生成器/客户端链。不能把未确认的兼容方案写成用户已接受。
- 是否采用其余推荐精确版本、镜像发行版和 digest；是否接受 Playwright runner 内置 Node patch 与项目 Node patch 独立记录。
- 在已确认的首轮 macOS arm64 / linux/arm64 验收范围内，固定具体宿主工具版本和验收记录字段；其他适配目标继续按既有原则标注待验证，不重开其非阻塞地位。
- Compose 支持下限与实际 Engine/CLI/Buildx 组合；区分已知语法能力、官方发布元数据和本地验证结果。
- 将本报告具体兼容取舍代入已确认调整边界：本次 TypeScript 工具链与可能的生成器替换先由用户决定；实施阶段按既定规则处理不改变支持边界的补丁、路径、权限和构建参数，越过技术、主版本、数据语义或验收边界时重新讨论。

这些取舍已经属于现有冻结票的问题范围，本研究不另建内容重复的决策票，不替用户完成冻结。
