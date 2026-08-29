---
kind: external_dependency
name: Cordis 插件框架（DeepSeek Harness 的插件运行时）
slug: cordis
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
source_files:
    - vendor/README.md
    - docs/cordis-primer.md
    - docs/architecture.md
---

### Cordis
- 角色：DeepSeek Harness (`dsh`) 内置的插件框架，所有功能（模型适配器、工具注册表、会话日志、agent loop 本身）都以插件形式挂载到共享 `ctx` 中；没有特权核心可打补丁，扩展通过向 `ctx.<key>` 注册服务、事件和可撤销 effect 完成。
- 关键行为约定：服务声明在 `ctx` 上暴露稳定 key（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），依赖通过 `inject` 表达加载顺序；事件分 emit/waterfall/parallel/serial/bail 五种模式；注册必须是可撤销 effect，卸载时自动回滚。
- 与 dsh 的关系：dsh 是 Cordis 之上的“应用层”，通过 profile + bundle 的有序 patch 层把多个 Cordis 插件组合成 web/headless/sdk/acp 等运行形态。