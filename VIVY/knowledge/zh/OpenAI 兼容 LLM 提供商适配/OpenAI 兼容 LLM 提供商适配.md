---
kind: external_dependency
name: OpenAI 兼容 LLM 提供商适配
slug: openai-compatible-model
category: external_dependency
category_hints:
    - vendor_identity
    - auth_protocol
scope:
    - '**'
---

通过 `github.com/cloudwego/eino-ext/components/model/openai` 接入 OpenAI 兼容的 LLM 服务，密钥通过环境变量 `OPENAI_API_KEY` 注入。Provider 适配位于 `internal/provider/openai.go`，对外暴露为 Vivy 统一的 Model Provider 接口，支持任意 OpenAI 兼容端点（含第三方兼容服务）。密钥不持久化（D-010），仅在运行时注入。