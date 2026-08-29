---
kind: build_system
name: Cargo Workspace + Justfile + GitHub Actions 多平台构建与发布流水线
category: build_system
scope:
    - '**'
source_files:
    - Cargo.toml
    - justfile
    - .github/workflows/ci.yml
    - scripts/package-windows-gui.ps1
    - scripts/package-macos.sh
    - scripts/package-linux.sh
    - scripts/ci/package_headless.py
    - agent-diva-gui/src-tauri/tauri.conf.json
    - scripts/feature-gate-check.py
    - scripts/feature-gate-check.ps1
    - scripts/ci/check_laputa_clean_break.py
    - scripts/ci/check_cognitive_clean_break.py
    - deny.toml
    - clippy.toml
    - rustfmt.toml
    - contrib/systemd/agent-diva.service
    - contrib/launchd/com.agent-diva.gateway.plist
---

## 1. 构建系统总览

Agent Diva 采用 **Rust Cargo workspace** 作为核心构建编排，配合 **justfile** 统一入口、**GitHub Actions** CI/Release 流水线，以及各平台专属打包脚本（Shell / PowerShell / Python），形成“源码 → 二进制 → 安装包”的完整链路。

- workspace 根 `Cargo.toml` 声明 `resolver = "2"`，集中管理 17 个 crate（core/agent/autodream/e2e/tooling/neuron/providers/channels/tools/files/cli/service/migration/gui/src-tauri/manager/laputa/sandbox）。
- 全局版本通过 `[workspace.package] version = "0.9.9"` 与 `rust-version = "1.80.0"` 锁定；所有子 crate 以 path 依赖引用共享 crate（如 `agent-diva-files = { version = "0.9.9", path = "agent-diva-files" }`）。
- 发布 profile 启用 `opt-level=3, lto=true, codegen-units=1, strip=true`，开发 profile 保留 debug 信息。

## 2. 关键文件与职责

| 文件 | 作用 |
|---|---|
| `Cargo.toml` (root) | workspace 成员、全局依赖、release/dev profile |
| `justfile` | 本地/CI 统一命令：`build`, `test`, `check`, `fmt-check`, `ci`, `feature-gate-check`, `laputa-clean-break-check`, `cognitive-clean-break-check`, `bml-boundary-check`, `health-benchmark-check`, `e7-automated-release-gate`, `msrv-probe`, `package-linux`, `cross-linux-x86_64`, `cross-linux-arm64`, `build-deb`, `build-macos-universal`, `build-macos-dmg`, `package-windows-gui`, `trigger-build` |
| `.github/workflows/ci.yml` | 三平台 Rust check/build、GUI Tauri 构建、可选 coverage、tag 触发 Release |
| `scripts/package-windows-gui.ps1` | Windows GUI 一键打包：cargo release → pnpm bundle:prepare → NSIS 预缓存 → tauri build |
| `scripts/package-macos.sh` | macOS 通用二进制（x86_64 + aarch64 via `lipo -create`）+ tar.gz + 可选 DMG (`create-dmg`) |
| `scripts/package-linux.sh` | Linux CLI 包：编译 + 复制 systemd 服务文件 + tar.gz + sha256sum |
| `scripts/ci/package_headless.py` | 生成 headless bundle：拷贝 binary + README + systemd/launchd 脚本 + 写 `bundle-manifest.txt` + zip/tar.gz |
| `agent-diva-gui/src-tauri/tauri.conf.json` | Tauri v2 配置：产物目标 `nsis, msi, app, dmg, deb, appimage`，资源目录 `resources/` |
| `contrib/systemd/`, `contrib/launchd/` | 系统服务单元与安装/卸载脚本，被打包进 headless bundle |
| `deny.toml`, `clippy.toml`, `rustfmt.toml` | 依赖审计、lint、格式化规则 |

## 3. 架构与约定

### 3.1 构建入口分层
- **本地开发**：`just <recipe>` 是单一入口。`just ci` 串联 `fmt-check → check → test → health-benchmark-check → feature-gate-check → laputa-clean-break-check → cognitive-clean-break-check → bml-boundary-check`，作为 E7 自动化发布门控。
- **CI**：`.github/workflows/ci.yml` 分两个 job：`rust-check`（ubuntu/windows/macos 矩阵，执行 `just fmt-check/check/build` 及 feature gate 检查）、`gui-build`（pnpm + tauri build 产出 AppImage/deb/dmg/msi/exe）。`coverage` job 仅在 workflow_dispatch 且开启 `run_coverage` 时运行 tarpaulin + Codecov。
- **发布**：push tag `v*.*.*` 触发 `release` job，下载三平台 GUI artifact 并通过 `softprops/action-gh-release` 创建 GitHub Release。

### 3.2 多 crate 特性门控验证
`just feature-gate-check` 调用 `scripts/feature-gate-check.py/.ps1`，逐个验证非默认 feature flag 可独立编译，防止 feature 组合破坏。

### 3.3 边界/契约静态检查
- `laputa-clean-break-check`：Python 脚本扫描代码，证明 Embedded Laputa 是唯一生产 Memory runtime。
- `cognitive-clean-break-check`：断言 D4 §3.1 遗留认知表面在生产路径中已删除。
- `bml-boundary-check`：运行 `agent-diva-laputa` 的 `bml_boundary_guard` 测试，禁止 governance 模块直接调用 BML 写入 API。
- `memory-provider-check`：聚焦 memory-provider 组装与失败回归测试。
- `health-benchmark-check`：运行 manager 的 `/api/health` benchmark 用例，确保 CI 预算内。

### 3.4 MSRV 隔离构建
`just msrv-probe` 使用 `cargo +1.80` 并在独立 `target/msrv-1.80` 目录下构建，避免污染默认 target，用于验证 `rust-version = "1.80.0"` 承诺。

### 3.5 跨平台打包策略
| 平台 | 工具链 | 产物 |
|---|---|---|
| Linux | `cross` (Docker) + `cargo-deb` + `cargo generate-rpm` | deb/rpm/zip |
| macOS | `lipo -create` 合并 x86_64/aarch64 + `create-dmg` | universal binary + tar.gz + DMG |
| Windows | `scripts/package-windows-gui.ps1` 预缓存 NSIS 工具链并校验 SHA1 | NSIS/MSI 安装包 + exe |
| Headless (CI) | `scripts/ci/package_headless.py` | zip (Windows) / tar.gz (Unix)，内含 systemd/launchd 脚本 |

### 3.6 GUI 构建管线
Tauri 前端基于 Vue 3 + Vite + pnpm，后端 Rust 在 `agent-diva-gui/src-tauri`。CI 步骤：`pnpm install --frozen-lockfile` → `pnpm tauri build`，产物上传到 `target/release/bundle/**/*.{AppImage,appimage,deb,dmg,msi,exe}`。Windows 本地打包脚本会先 `cargo build --release -p agent-diva-cli -p agent-diva-service`，再 `pnpm bundle:prepare` 将二进制注入 `src-tauri/resources/manifests/gui-bundle-manifest.json`，最后 `pnpm tauri build`。

## 4. 约定与约束

- **统一入口**：所有构建/测试/打包必须通过 `just` recipe 或 CI 脚本调用，禁止直接散跑 `cargo build`（CI 中显式 `just fmt-check/check/build`）。
- **版本来源单一**：workspace 根 `Cargo.toml` 中的 `version` 是唯一真实值；子 crate 不单独声明版本，仅用 `{ version = "0.9.9", path = ... }` 引用。
- **CI 最小化**：push/PR 默认不跑全量 `cargo test --all`，需 `workflow_dispatch` 且 `run_tests=true` 才执行；覆盖率同理需 `run_coverage=true`。
- **安全构建**：release profile 强制 strip + LTO；Windows GUI 打包脚本对下载的 NSIS zip 和 `nsis_tauri_utils.dll` 做 SHA1 校验，不匹配即抛错。
- **产物精简**：CI 上传 artifact 仅匹配最终安装包后缀（`.AppImage/.deb/.dmg/.msi/.exe`），避免把 AppDir/GTK 中间产物挂到 Release。
- **服务集成**：headless bundle 自动附带 `contrib/systemd` 或 `contrib/launchd` 下的 service/unit 与 install/uninstall 脚本，由 `package_headless.py` 写入 manifest 并标记为可执行。
- **跨编译要求**：Linux 交叉编译需 Docker + `cross`；macOS DMG 需 `brew install create-dmg`；Windows GUI 需预先安装 `pnpm`、`cargo`、`python`，NSIS 工具链由脚本自动预缓存至 `%LOCALAPPDATA%\tauri\NSIS`。
- **格式与 lint 门禁**：`just check` 等价于 `cargo clippy --all -- -D warnings`，`just fmt-check` 等价于 `cargo fmt --all -- --check`，CI 中二者均作为必过项。