---
kind: dependency_management
name: Rust Workspace + pnpm 双栈依赖管理（集中版本、Cargo Lock、Deny 合规与 crates.io 发布校验）
category: dependency_management
scope:
    - '**'
source_files:
    - Cargo.toml
    - Cargo.lock
    - deny.toml
    - agent-diva-gui/package.json
    - agent-diva-gui/pnpm-lock.yaml
    - agent-diva-gui/shared-avatar-protocol/package.json
    - agent-diva-gui/avatar-runtime-vrm/package.json
    - scripts/wait-crates-io-version.sh
    - agent-diva-agent/Cargo.toml
    - agent-diva-cli/Cargo.toml
---

## 1. 使用的系统/方法

本仓库采用 **双栈依赖管理**：
- Rust 侧使用 Cargo workspace（`Cargo.toml` `[workspace]`），通过 `workspace.dependencies` 集中声明所有子 crate 共享的第三方依赖，并以根级 `Cargo.lock` 锁定全部 crate 版本。
- GUI 侧使用 pnpm（`agent-diva-gui/package.json` 中声明 `packageManager: "pnpm@10.33.2+sha512..."`），并通过 `pnpm-lock.yaml` 锁定前端依赖；同时通过 `file:` 协议在 `agent-diva-gui/shared-avatar-protocol` 与 `avatar-runtime-vrm` 两个本地 npm 包之间建立内聚引用。
- 安全与许可证合规由 `deny.toml`（cargo-deny）在 CI 或本地执行 `cargo deny check licenses/advisories/bans/sources` 强制约束。
- 发布到 crates.io 的流程通过 `scripts/wait-crates-io-version.sh` 轮询 crates.io API，确保发布的 semver 版本已被索引后再继续后续步骤。

## 2. 关键文件

- `Cargo.toml` — workspace 定义、`[workspace.package]` 统一 version/rust-version/authors、`[workspace.dependencies]` 集中声明所有第三方依赖。
- `Cargo.lock` — 全工作区锁文件，锁定所有传递依赖的确切版本。
- `deny.toml` — cargo-deny 配置：限定目标 triple、白名单/黑名单许可证、advisory 策略、仅允许 `crates.io-index` 源。
- `agent-diva-gui/package.json` + `pnpm-lock.yaml` — 前端依赖声明与锁定。
- `agent-diva-gui/shared-avatar-protocol/package.json` / `agent-diva-gui/avatar-runtime-vrm/package.json` — 本地 npm 子包，通过 `file:` 引用彼此。
- `scripts/wait-crates-io-version.sh` — 发布后等待 crates.io 索引完成的脚本。
- 各子 crate 的 `Cargo.toml`（如 `agent-diva-agent/Cargo.toml`、`agent-diva-cli/Cargo.toml` 等）—— 以 `{ path = "../xxx", version = "0.9.9" }` 形式引用同 workspace 内的其他 crate。

## 3. 架构与约定

### 3.1 Rust workspace 集中化依赖
- 所有跨 crate 复用的第三方库（tokio、serde、reqwest、tracing、clap、sqlx、uuid、regex 等）均在根 `Cargo.toml` 的 `[workspace.dependencies]` 中以固定主版本声明，子 crate 通过 `workspace = true` 或直接写版本号引用，保证同一依赖在工作区内版本一致。
- 子 crate 之间的内部依赖统一通过 `path = "../xxx"` 加 `version = "0.9.9"` 双重声明，既满足本地开发路径解析，又保持发布时语义化版本契约。
- 工作区统一 `rust-version = "1.80.0"`，避免不同 crate 对工具链版本产生分歧。
- 发布 profile 统一启用 `opt-level=3`、`lto=true`、`codegen-units=1`、`strip=true`，保证二进制体积与性能一致。

### 3.2 前端依赖（pnpm）
- GUI 使用 pnpm 作为包管理器，并在 `package.json` 中通过 `packageManager` 字段锁定 pnpm 版本及 sha512 integrity，确保构建可重现。
- 本地子包 `shared-avatar-protocol` 与 `avatar-runtime-vrm` 通过 `file:./...` 相互引用，不发布到 npm registry。
- 外部依赖使用 `^` 或 `~` 范围声明（如 `vue ^3.5.13`、`@tauri-apps/api ~2.11.1`），实际锁定由 `pnpm-lock.yaml` 维护。

### 3.3 依赖来源与合规
- `deny.toml` 的 `[sources]` 段仅允许 `https://github.com/rust-lang/crates.io-index`，禁止从未知 registry/git 拉取 crate，从而将 Rust 依赖来源收敛到 crates.io。
- 许可证策略：MIT、Apache-2.0、BSD-2/3-Clause、ISC、Unicode-DFS-2016、MPL-2.0、OpenSSL 为允许列表；GPL-2.0/GPL-3.0 明确拒绝；未检测到许可证的 crate 直接 deny。
- advisories 策略：`vulnerability = "deny"`，`unmaintained/yanked/notice = "warn"`，且 `severity_threshold = "low"`，即低及以上严重性漏洞都会阻断构建。
- bans 策略：同一 crate 多个版本会发出 warn，wildcards 允许。

### 3.4 发布流程中的依赖校验
- `scripts/wait-crates-io-version.sh` 通过 curl 轮询 `https://crates.io/api/v1/crates/${CRATE}`，直到返回的 JSON 中包含指定 `num == $VERSION` 的版本条目才退出，用于在 CI 中等待 crates.io 完成索引后再进行下游操作。

## 4. 约定与约束

- **workspace 内版本同步**：所有子 crate 的 `version` 均设为 `0.9.9`（来自 `[workspace.package]`），内部 crate 间依赖同时声明 `path` 与 `version`，形成“开发用 path + 发布用 semver”的双轨约定。
- **第三方依赖不得在各子 crate 重复声明**：应通过根 `Cargo.toml` 的 `[workspace.dependencies]` 统一管理，避免版本漂移。
- **禁止引入非 crates.io 的 Rust 依赖**：`deny.toml` 的 `allow_registry` 仅包含 crates.io-index，未知 registry/git 来源会触发 warn/deny。
- **许可证白名单强制**：任何不在 allow 列表的许可证都会被 `cargo deny check licenses` 拒绝，新增依赖需先确认其许可证是否在白名单。
- **GUI 前端依赖必须经 pnpm-lock.yaml 锁定**：新增/升级前端依赖后需提交 `pnpm-lock.yaml`，以保证 CI 可重现。
- **发布前需通过 cargo-deny**：CI 应在打包前运行 `cargo deny check licenses advisories bans sources`，否则视为失败。
- **发布 crates.io 版本需等待索引完成**：发布流程调用 `wait-crates-io-version.sh`，若 crates.io 尚未索引该版本则阻塞后续步骤。

## 5. 适用性说明

本仓库存在完整的 Rust workspace 依赖管理（`Cargo.toml` + `Cargo.lock`）、前端 pnpm 依赖管理（`package.json` + `pnpm-lock.yaml`）、cargo-deny 合规策略以及 crates.io 发布校验脚本，因此本类别完全适用。