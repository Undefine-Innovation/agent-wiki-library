---
kind: build_system
name: pnpm Workspace + TypeScript/TSX 构建与多平台发布流水线
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - scripts/build.ts
    - scripts/build-exe-for-python-sdk.ts
    - .gitlab-ci.yml
    - scripts/release/bump.ts
    - vitest.config.ts
    - apps/cli/package.json
    - tsconfig.base.json
    - tsconfig.host.json
    - tsconfig.client.json
---

## 1. 使用的系统与方法

仓库采用 **pnpm workspace**（`pnpm-workspace.yaml`）作为多包工作区根，聚合 `packages/*/*`、`native/landlock-run`、`apps/*`、`website`、`python/sdk-runtime` 等子模块。所有构建、测试、文档生成、发布均通过根 `package.json` 的 `scripts` 统一编排，核心入口为 `tsx scripts/build.ts`。

- **TypeScript 编译**：使用 `tsc -b` 分别产出 host (`tsconfig.host.json`) 与 client (`tsconfig.client.json`) 两套产物，再通过 `tsdown --env.DSH_BUILD_FACE <host|client>` 做二次打包。
- **Web 前端构建**：`apps/web` 基于 Vite/VitePress，由 `pnpm --filter @deepseek-ai/dsh-web-frontend run build` 触发。
- **测试框架**：Vitest，通过根 `vitest.config.ts` 定义 thread-safe / process-bound 双项目，配合 `vitest.e2e.config.ts`、`vitest.expected.config.ts`、`vitest.snapshot.config.ts`、`vitest.web.config.ts`、`vitest.web.perf.config.ts`、`vitest.web-stress.config.ts` 等多套配置。
- **覆盖率门禁**：V8 provider，强制 per-file 100%（语句/分支/函数/行），通过 `scripts/run-gates.ts` 分阶段执行 `check:ci:*` 门禁。
- **Python SDK 发布**：`.gitlab-ci.yml` 中通过 `scripts/build-python-release.py` 与 `scripts/build-exe-for-python-sdk.ts` 将 Node 运行时用 `@yao-pkg/pkg --sea` 打包为单文件可执行体，再封装为 wheel 发布到 GitLab PyPI。
- **版本管理**：`scripts/release/bump.ts` 维护两个发布家族——`dsh`（共享版本号，workspace root 与各 publishable package 同步）和 `vendor`（每个 vendored 包独立递增）。CI 禁止写回仓库，仅人类在合并后打 tag。

## 2. 关键文件

- `package.json`：工作区入口，声明 `build`、`build:lib`、`build:web`、`test:*`、`check:ci:*`、`release:*`、`docs:*` 等全部脚本。
- `pnpm-workspace.yaml`：工作区成员、`linkWorkspacePackages`、`overrides`、`peerDependencyRules`、`allowBuilds`（白名单策略）、`patchedDependencies`（`node-pty` patch）。
- `scripts/build.ts`：统一入口，解析 `--profile` / `DSH_BUILD_CLIENT_PROFILE`，调用 `build:lib` → `build:web`，写入客户端构建记录。
- `scripts/build-exe-for-python-sdk.ts`：Python SDK 运行时单文件可执行体构建管线（verify closure → pnpm build → deploy staging → inject pkg config → `pkg --sea` → 复制 ripgrep/native addon → 同步至 `python/sdk-runtime/src/deepseek_harness_runtime/runtime`）。
- `.gitlab-ci.yml`：GitLab CI 两阶段（build / publish），按 `python-v<version>` tag 触发，构建 linux-x64/arm64、macos-arm64、win-x64 四平台 runtime wheel，并通过 `twine upload` 发布。
- `vitest.config.ts`：全局 Vitest 配置，含平台排除、进程隔离、per-file 100% 覆盖率阈值、覆盖豁免清单。
- `scripts/release/bump.ts`：版本递增与提交逻辑，区分 dsh/vendor 家族。
- `apps/cli/package.json`：`dsh` CLI 包，`bin.dsh = lib/bin.js`，依赖大量 `@deepseek-ai/dsh-*` workspace 包。
- `tsconfig.base.json` / `tsconfig.host.json` / `tsconfig.client.json`：跨包 TS 路径映射与编译目标。

## 3. 架构与约定

- **分层构建**：`build:lib:host` 先产出 host 侧 `lib/`，`build:lib:client` 产出 client 侧；`build:web` 单独构建前端；最终 `build` 串联两者并记录公开环境变量。
- **闭包验证**：Python SDK 发布前必须通过 `pnpm run verify-runtime-closure`，确保部署闭包无符号链接、包含所有 Cordis 运行时 bare-package import 所需的资源。
- **平台矩阵**：CI 显式声明 `linux-x64`、`linux-arm64`、`macos-arm64`、`win-x64` 四个 target triple（`node24-linux-x64` 等），Windows 仅支持 x64；Linux 产物要求 glibc ≤ 2.28（manylinux_2_28）。
- **安全约束**：`pnpm-workspace.yaml` 的 `allowBuilds` 默认拒绝所有带 install/build script 的依赖，仅允许 esbuild、lefthook、node-pty、koffi 等经审计的包；`minimumReleaseAgeExclude` 精确放行已审查的第三方二进制包。
- **测试分区**：Vitest 将 suite 分为 thread-safe 与 process-bound 两类，后者因涉及进程全局状态而隔离运行；Windows 下额外跳过 bash/sandbox 相关套件。
- **发布流程**：`release:dsh` 接受 `major|minor|patch|x.y.z`，`release:vendor` 自动递增各 vendored 包版本；CI 仅在 `python-v<version>` tag 时触发 Python 发布，且 tag 必须与 `package.json` version 严格一致。

## 4. 约定与约束

- **Node 引擎锁定**：根 `package.json` 的 `engines.node` 限定 `^22.19.0 || >=24.0.0`，构建脚本默认使用 node24 目标。
- **pnpm 版本锁定**：`packageManager: "pnpm@11.7.0"`，CI 通过 `corepack enable` 复用。
- **覆盖率 100% 门禁**：`vitest.config.ts` 中 `thresholds.perFile = true` 且四项指标均为 100；任何未覆盖路径都会导致合并失败，豁免需显式列入配置并附原因注释。
- **构建产物不可变**：`scripts/release/bump.ts` 注释明确“CI 从不写回仓库”，版本变更通过本地 bump + git commit + 人工打 tag 完成。
- **Python 运行时闭包完整性**：`build-exe-for-python-sdk.ts` 在打包前清理 staging 目录、恢复 legacy hoist、物化所有符号链接，并校验 `node-pty` 原生 addon 存在；macOS 额外复制 `spawn-helper` 并检查部署目标。
- **依赖安装白名单**：`pnpm-workspace.yaml` 的 `allowBuilds` 以 deny-by-default 策略限制原生构建，新增依赖若带 lifecycle script 必须显式加入白名单。
- **快照测试**：`test:expected` 与 `test:snapshot` 通过 `DSH_SNAPSHOT=refresh` 或 `record` 模式更新预期输出，用于 ACP/SDK/headless 会话回放断言。
- **文档站点构建**：`docs:build` 调用 `tsx website/build.ts` 并随后运行 `verify-doc-site-fragments` 校验片段一致性。