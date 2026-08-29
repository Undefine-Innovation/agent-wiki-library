---
kind: configuration_system
name: Cordis 插件配置系统：Profile + Patch 分层与 Schemastery 校验
category: configuration_system
scope:
    - '**'
source_files:
    - packages/boot/app-boot/src/index.ts
    - packages/boot/app-boot/src/profile.ts
    - apps/cli/src/bin.ts
    - apps/cli/src/profile-boot.ts
    - apps/cli/src/dump-config.ts
    - scripts/gen-config-catalog.ts
    - docs/config-catalog.md
    - scripts/verify-config-source-ownership.ts
    - scripts/verify-cordis-config.ts
---

## 1. 使用的系统与框架

DeepSeek Harness 使用自研的 **Cordis** 插件框架（`@deepseek-ai/cordis`）作为运行时装配核心，配合 `@deepseek-ai/cordis-plugin-loader`、`@deepseek-ai/cordis-plugin-include`、`@deepseek-ai/cordis-plugin-group` 实现插件发现、YAML 配置加载、分组隔离与 patch 覆盖。每个可配置能力以独立 npm 包（如 `dsh-agent-loop`、`dsh-bash-local`、`dll-deepseek` 等）暴露一个 TypeScript `Config` 接口，并通过 **Schemastery** schema 在启动时做字段级校验。配置声明通过 `scripts/gen-config-catalog.ts` 扫描源码生成 `docs/config-catalog.md`，并由 `verify-config-catalog` 保证 schema 与声明类型一一对应。

## 2. 关键文件与包

- `packages/boot/app-boot/src/index.ts`：应用启动胶水，提供 `loadLayeredEnv`、`boot`、`mountRootInclude`、`loadOptionalPatches`、`loadOverlayPatches`、`renderConfigDump`、`watchUserPatches`、`installFailLoud`、`assertEntriesActivated` 等核心 API。
- `packages/boot/app-boot/src/profile.ts`：Profile 管理——解析 `$DSH_HOME/profiles/<name>`，读取 `package.json` 中的 `dsh.profile.bundles`，按顺序装载 bundle 的 `cordis.patch.yml`，再叠加用户层 `cordis.patch.yml`；维护 `profiles/node_modules` 模块回退（symlink / ESM proxy）。
- `apps/cli/src/bin.ts`：CLI 入口，调用 `parseDshArgs` 后根据 mode (`profile`/`plugin`/`dump-config`) 分发。
- `apps/cli/src/profile-boot.ts`：组合 profile 的 patch 栈（bundle patches → profile patch → home patch → `--patch` overlays → telemetry switch），并安装 HMR watcher。
- `apps/cli/src/dump-config.ts`：`dsh --dump-config` 的实现，打印带来源注释的可复现 YAML。
- `scripts/gen-config-catalog.ts`：从各包入口扫描 `Config` 接口与 JSDoc，生成配置目录文档。
- `scripts/verify-config-source-ownership.ts`、`scripts/verify-cordis-config.ts`：构建期校验配置文件归属与 schema 一致性。
- `docs/config-catalog.md`：由脚本生成的全部插件配置声明参考（含依赖服务键、JSDoc、source 指针）。

## 3. 架构与约定

### 3.1 Profile + Patch 分层模型

启动路径为 `dsh --profile <name>`，其有效配置是以下层的**有序叠加**（后层优先）：

1. **Bundle layers**：`package.json` 中 `dsh.profile.bundles` 列出的包，按顺序装载各自 `dsh.bundle.patch` 指向的 `cordis.patch.yml`。
2. **Profile 用户层**：`$DSH_HOME/profiles/<name>/cordis.patch.yml`。
3. **Home 用户层**：`$DSH_HOME/cordis.patch.yml`（机器级偏好，优先级高于 profile 层）。
4. **Overlay layers**：命令行 `--patch <file>` 传入的文件，按 argv 顺序依次叠加。
5. **Flag-derived patch**：如 `DSH_TELEMETRY_DISABLED` 被设为非空值时，注入一条禁用 `session-telemetry-otel` 行的 patch。

所有层都是 `PatchOptions[]`（id 目标覆盖、`disable`、`insert` 列表），通过 `applyEntryPatches` 一次性扁平化合并，避免多次遍历导致的状态漂移。根条目列表始终是一个空的 `cordis.yml`（`PROFILE_ROOT_CONFIG = '[]'`），真实组成完全由 patch 堆叠产生。

### 3.2 环境变量分层与保护

`loadLayeredEnv('dsh', cwd)` 按三层顺序组装不可变快照：

| 层级 | 来源 | 说明 |
|---|---|---|
| `process` | 继承进程环境 | 最高优先级 |
| `project-env` | 当前工作目录 `.env` | 仅当该变量不存在时才写入 |
| `user-env` | `$DSH_HOME/.env` | 仅当该变量不存在时才写入 |

`.env` 中禁止设置“引导类”变量名（`PATH`、`HOME`、`NODE_OPTIONS`、`SSL_CERT_*`、`HTTP_PROXY*`、`DEEPSEEK_BASE_URL` 以及所有 `DSH_`、`XDG_`、`DYLD_`、`BASH_FUNC_` 前缀），这些只能由进程启动时注入，防止 `.env` 劫持进程或网络引导行为。最终返回 `LaunchEnvironmentSnapshot`，供插件通过 `ctx.get(DSH_LAUNCH_ENVIRONMENT_KEY)` 读取。

### 3.3 Cordis Loader 挂载流程

`boot(binName, absoluteConfigPath, patches?, prepare?, bareModuleBaseUrl?)` 执行：

1. 创建 `Context`，注册 `dshHomePath` 解析器到上下文。
2. 安装 `Loader` 服务。
3. 可选地运行 `prepare(ctx)`（此时尚无配置树 entry 挂载）。
4. 通过 `mountRootInclude` 将 `cordis:include` 内置项挂载到指定 `cordis.yml`，并应用初始 patches。
5. 等待 `loader.await()` 使所有 entry fiber 进入终态。
6. 调用 `assertEntriesActivated` 审计：所有 enabled entry 必须处于 `ACTIVE` 状态，否则抛出包含缺失 service 名称的诊断。
7. 返回根 `Context`。

未处理的异步拒绝由 `installFailLoud` 捕获，输出 stderr 并在最多 2 秒内等待 release hook（用于恢复终端模式）后 `exit(1)`。

### 3.4 运行时热重载

对标记 `patchReload: 'live'` 的 profile（默认自定义 profile），`watchUserPatches` 通过 Cordis HMR 监听 profile 和 home 的 `cordis.patch.yml`，每次变更重新调用 `composeLive()` 生成新的 patch 列表，并以事务方式调用 `entry.update({ config })` 替换 include 的 patches。bundle 层位于用户层之下，因此用户编辑永远无法覆盖 bundle 插入的行。

### 3.5 配置声明与校验

每个插件包的 `Config` 接口即部署轴上的配置契约。`gen-config-catalog.ts` 用 TypeScript 编译器 AST 扫描各包入口，提取接口声明、JSDoc、引用类型，并与 Schemastery schema 交叉校验：schema 中出现的每个 key（含嵌套路径）必须在声明类型上可定位，从而保证“声明即 schema”。生成的 `docs/config-catalog.md` 同时列出每个配置块 `Requires:` 的服务键（插件 `inject`s 的 service name），部署者必须确保这些服务也被加载。

### 3.6 Agent Preset 配置

`agent-presets` 包定义 `Config { default, roots[], includeShippedRoot, includeUserRoot }`，其中 `roots` 是扫描 preset 子目录的路径数组，每个 root 附带 `trust: 'system' | 'user'`。`system` preset 来自部署打包，`user` preset 来自 `$DSH_HOME/USER_PRESET_DIR`，后者信任级别等同于 shell 访问。preset id 必须符合正则 `/^[a-z0-9][a-z0-9-]*$/`，作为路径段进行安全约束。

## 4. 约定与约束

- **配置即补丁**：所有 `cordis.yml` 内容都通过 `PatchOptions` 表达（id 覆盖、`disable`、`insert`），没有“直接写死”的配置行；这保证了同一份 composition 可在不同部署间 diff。
- **分层顺序固定**：bundle → profile → home → overlay → flag-derived，任何新层必须显式插入此序列，不能自行插队。
- **`.env` 白名单**：`BOOTSTRAP_NAMES` 与 `BOOTSTRAP_PREFIXES` 定义的变量名绝对禁止出现在 `.env` 中，违反时直接抛错。
- **Schema 强校验**：插件 `Config` 必须配套 Schemastery schema；`verify-config-catalog` 会在 CI 中检查 schema 与声明类型同步。
- **诊断失败即退出**：`assertEntriesLoaded` 与 `assertEntriesActivated` 在启动后审计，任何未激活的 enabled entry 都会抛出错误；`installFailLoud` 保证未处理拒绝不会静默吞掉。
- **可复现 dump**：`renderConfigDump` 输出带 `# == <origin>, patched by ...` 注释的 YAML，每一行都标注来源文件和覆盖它的 layer，可用于调试配置冲突。
- **Profile 模板化**：内置 `acp`、`web`、`headless`、`sdk`、`sdk-minimal` 五种 profile 模板，首次使用时自动初始化 `package.json`、`cordis.patch.yml`、`pnpm-workspace.yaml`。
- **模块回退**：`healProfilesModuleFallback` 在 `$DSH_HOME/profiles/node_modules` 中以 symlink（plain Node）或 ESM proxy（打包可执行）形式镜像 dsh 安装的依赖闭包，保证 out-of-tree plugin 能解析 peer 依赖。
- **配置表达式**：patch 文件支持 `!!js` 标量表达式，可在运行时求值并访问 `process.env`，但表达式本身在 dump 中按原样打印，不执行。
- **Telemetry 开关**：`DSH_TELEMETRY_DISABLED` 为非空字符串时，无论值是什么都禁用 `session-telemetry-otel` 行，遵循“隐私开关宁可误关也不误开”的原则。
- **配置所有权验证**：`verify-config-source-ownership.ts` 在构建期检查每个配置键的来源是否属于声明它的那个包，防止跨包污染配置空间。