---
kind: dependency_management
name: pnpm 工作区 + vendor 源码级内联 + Dependabot/uv 联合依赖治理
category: dependency_management
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - pnpm-lock.yaml
    - vendor/README.md
    - .github/dependabot.yml
    - patches/node-pty@1.2.0-beta.15.patch
    - python/sdk/pyproject.toml
    - python/sdk-runtime/pyproject.toml
---

## 1. 使用的系统与工具

- **包管理器**：pnpm（根 `package.json` 通过 `packageManager: "pnpm@11.7.0"` 锁定版本，Node 引擎要求 `^22.19.0 || >=24.0.0`）。
- **工作区**：`pnpm-workspace.yaml` 声明了 6 类 workspace：`vendor/*`、`packages/*/*`、`native/landlock-run` 及其子包、`apps/*`、`website`、`python/sdk-runtime`。通过 `linkWorkspacePackages: true` 让本地构建把同名包解析到本仓库的 pinned 源。
- **锁文件**：根目录 `pnpm-lock.yaml`（lockfileVersion 9），所有 npm 依赖的版本与链接关系由它固化。
- **Python 侧**：`python/sdk` 使用 `pyproject.toml` + `uv.lock`（见目录树中的 `uv.lock`），构建后端为 hatchling；`python/sdk-runtime` 同样用 `pyproject.toml` 打包二进制产物。
- **自动更新**：`.github/dependabot.yml` 同时监控 npm（排除 `vendor/**`）、`python/sdk` 下的 uv 以及 GitHub Actions。
- **补丁机制**：`patches/node-pty@1.2.0-beta.15.patch` + `pnpm-workspace.yaml#patchedDependencies` 实现 pnpm patch-package 风格的补丁。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `package.json` | 根工作区入口，声明 workspaces、scripts、devDependencies、`packageManager`、`engines` |
| `pnpm-workspace.yaml` | 工作区成员、`overrides`、`peerDependencyRules`、`allowBuilds`、`minimumReleaseAgeExclude`、`patchedDependencies` |
| `pnpm-lock.yaml` | 全仓依赖锁定（含 workspace link 与 vendored 包映射） |
| `vendor/README.md` | 源码级 vendoring 的完整规范、manifest 表、修改日志与同步流程 |
| `.github/dependabot.yml` | Dependabot 对 npm / uv / actions 的定时更新策略 |
| `patches/node-pty@1.2.0-beta.15.patch` | 针对 node-pty 的精确补丁 |
| `python/sdk/pyproject.toml` | Python SDK 声明依赖 `deepseek-harness-runtime-bin==0.0.0.dev0`，并通过 `[tool.uv.sources]` 以 editable 方式指向本地 `../sdk-runtime` |
| `python/sdk-runtime/pyproject.toml` | 运行时二进制包的发布配置，仅打包注入的 dsh 可执行文件 |

## 3. 架构与约定

### 3.1 三层依赖模型

1. **workspace 内部包**：`packages/*/*`、`apps/*`、`website`、`native/landlock-run/packages/*` 之间通过 `workspace:^` 引用，安装时解析为 `link:`，保证多包间类型与行为一致。
2. **vendored 框架层**：Cordis 框架及基础库被源码级复制到 `vendor/`，并重新 scoped 到 `@deepseek-ai/cosmokit`、`@deepseek-ai/schemastery`、`@deepseek-ai/cordis` 等名称。根 `pnpm-workspace.yaml` 通过 `overrides` 把这些命名空间强制 link 到本地 `vendor/*` 目录，从而“冻结”框架层版本，使其可审计、可打补丁、可独立升级。`verify-vendored-links` 门禁会断言 lockfile 中这些名称全部解析为 workspace link，不得出现 registry 副本。
3. **上游 npm 依赖**：除 vendored 框架外的第三方包走正常 npm 解析，由 `pnpm-lock.yaml` 锁定。

### 3.2 供应链安全策略

- `allowBuilds` 白名单：pnpm 10+ 默认拒绝任何带 install/build 脚本的依赖，只有显式列入的包（esbuild、lefthook、node-pty、koffi、特定 subprocess-local 包）允许运行脚本；其余如 `@google/genai`、`protobufjs`、`node-addon-require-builtin` 被明确标记为 `false`，即使它们作为可选依赖被拉入也不会执行生命周期脚本。
- `minimumReleaseAgeExclude`：对近期发布的平台专用包（如 `@anthropic-ai/claude-agent-sdk-*`、`@openai/codex` 各平台 alias）做精确版本豁免，绕过 pnpm 的 release-age 阻塞策略。
- Dependabot 对 `vendor/**` 路径排除，避免覆盖人工维护的 vendored 同步流程。

### 3.3 Python 依赖管理

- `python/sdk` 通过 `uv.lock` 锁定 Python 依赖，并在 `[tool.uv.sources]` 中以 `editable = true` 的方式将 `deepseek-harness-runtime-bin` 指向本地 `../sdk-runtime`，使开发期无需发布即可安装。
- `python/sdk-runtime` 是纯发布包，不声明运行时依赖，只通过 hatch 钩子把预编译的 `deepseek-harness-sdk-runtime-*` 二进制打入 wheel artifacts。

### 3.4 版本与发布约束

- 根 `package.json` 的 `engines.node` 限制 Node 版本范围。
- `pnpm-workspace.yaml#peerDependencyRules.allowedVersions.typescript` 限定 TypeScript 必须 `>=5 <7`，防止工作区内不同包引入冲突版本。
- 每个 vendored 包在 `vendor/README.md` 的 manifest 表中记录上游 commit hash，升级时必须同步更新版本号与哈希。

## 4. 约定与约束

- **vendored 包不可通过 npm 升级**：Dependabot 已排除 `vendor/**`；更新需按 `vendor/README.md` 的 “Sync procedure” 手动从上游 fork 拷贝源码、重 apply 本地修改、更新 manifest 表后 `pnpm install && pnpm run test && pnpm run build`。
- **新增 vendored 包**：遵循 `docs/cookbook/adding-a-vendored-package.md` 的流程，并把新包加入 `pnpm-workspace.yaml` 的 overrides 与 `vendor/README.md` 的 manifest 表。
- **不允许未审查的构建脚本**：任何新增依赖若携带 lifecycle script，必须在 `allowBuilds` 中显式声明（true/false），否则安装失败。
- **vendored 名称必须解析为 workspace link**：`verify-vendored-links` 会在 CI 中检查 lockfile，禁止出现同名包从 registry 拉取副本。
- **node-pty 补丁固定**：补丁版本与文件名硬编码在 `pnpm-workspace.yaml#patchedDependencies`，升级该包时需同步更新补丁。
- **Python 侧 editable 绑定**：`python/sdk` 通过 uv sources 的 `path + editable` 绑定本地 `sdk-runtime`，发布前需先构建 runtime 二进制；编辑安装不会冻结 wheel 快照。
- **依赖更新节奏**：Dependabot 统一在每天 04:00 (Asia/Shanghai) 触发，npm 与 uv 均设置 30 天 cooldown，避免 PR 风暴。
- **workspace 成员边界**：`pnpm-workspace.yaml` 显式列出所有 workspace 路径，新增包必须加入此列表才能被根脚本发现；当前未包含 `snapshots/`、`scripts/`、`docs/`，说明它们是脚本/文档而非可发布包。

## 5. 相关脚本（来自根 package.json scripts）

- `rescope-vendor` / `rescope-vendor:check`：对 vendored 包进行 `@deepseek-ai` 命名空间重映射。
- `gen-third-party-notices` / `verify-third-party-notices`：生成/校验第三方许可通知。
- `verify-dsh-package-licenses`：校验 DSH 包许可证合规性。
- `verify-optional-dependency-imports`：验证可选依赖仅在条件分支中被 import。
- `verify-runtime-closure` / `verify-built-package-invariants`：校验运行时闭包与构建产物不变量。
- `release:*`：bump/pack/verify/publish 发布流水线。
- `check:all` / `hygiene`：聚合质量门禁，其中包含依赖相关检查。