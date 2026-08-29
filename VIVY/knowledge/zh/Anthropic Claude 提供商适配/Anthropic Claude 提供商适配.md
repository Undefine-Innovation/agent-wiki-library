---
kind: external_dependency
name: Anthropic Claude 提供商适配
slug: anthropic-claude
category: external_dependency
category_hints:
    - vendor_identity
    - auth_protocol
scope:
    - '**'
---

Anthropic Claude 作为另一个 LLM 提供商适配，密钥通过环境变量 `ANTHROPIC_API_KEY` 注入。Provider 实现位于 `internal/provider` 下，遵循与 OpenAI 相同的抽象接口。密钥不持久化，仅运行时注入。