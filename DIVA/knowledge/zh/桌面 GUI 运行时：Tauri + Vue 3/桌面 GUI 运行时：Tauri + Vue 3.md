---
kind: external_dependency
name: 桌面 GUI 运行时：Tauri + Vue 3
slug: tauri
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
source_files:
    - agent-diva-gui/package.json
    - agent-diva-gui/src-tauri/Cargo.toml
    - README.md
---

Agent Diva 的桌面客户端基于 Tauri（Rust 后端）+ Vue 3 前端，GUI crate 通过 `pnpm tauri dev` / `pnpm tauri build` 启动与打包；开发模式默认由 Tauri 内嵌启动 Gateway，独立调试时可通过 `AGENT_DIVA_EXTERNAL_GATEWAY=1` 切换为外部 Gateway。Windows 发布使用 `scripts/package-windows-gui.ps1` 生成 NSIS/MSI 安装包，Linux 通过 `just package-linux` / `just build-deb` 打包。该运行时是 GUI 层唯一依赖的外部平台。