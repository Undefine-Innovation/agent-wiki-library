---
kind: external_dependency
name: Eino Agent 执行框架（Vivy 内核依赖）
slug: cloudwego-eino
category: external_dependency
category_hints:
    - framework_behavior
    - client_constraint
scope:
    - '**'
---

Vivy 物种内核基于 CloudWeGo Eino v0.9.13 构建，作为单 Go 模块的唯一 Agent 运行时框架。集成要点：
- 仅 `internal/runtime` 与 `internal/provider` 两个包可依赖 `github.com/cloudwego/eino*`；`domain`、`rpc`、`tools`、`storage`、`app` 等上层包不得反向引用 eino 类型，这是 PRD D-007 架构红线。
- `internal/provider` 通过 `eino-ext/components/model/openai` 适配 OpenAI 兼容模型，Anthropic 与 mock provider 也在此层实现，对外暴露为 Vivy 自有的 Provider 契约。
- `internal/runtime` 承担 Run 服务、Eino graph 适配、事件映射、中断、审批/预算/沙箱等编排逻辑。
- 参考源码 `.workspace/eino` 只读，本仓库消费 Eino 仅通过线上 go.mod 依赖。
- 验证记录见 `docs/eino-capability-verify.md`（checkpoint-bridge GO 能力验证 A1）。
注意：新增 provider 或 runtime 组件时必须遵守 eino 类型不泄漏到 UI 契约的约束。