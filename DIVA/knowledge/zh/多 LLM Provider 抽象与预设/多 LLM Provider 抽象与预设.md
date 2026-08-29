---
kind: external_dependency
name: 多 LLM Provider 抽象与预设
slug: llm-providers
category: external_dependency
category_hints:
    - vendor_identity
    - client_constraint
scope:
    - '**'
source_files:
    - agent-diva-providers/src/lib.rs
    - README.md
---

`agent-diva-providers` 提供统一的 LLMProvider trait 及 OpenAI 兼容客户端、Ollama 本地推理等实现，内置 45+ provider preset（OpenRouter、DeepSeek、OpenAI、Anthropic、Gemini、Zhipu、Moonshot、StepFun、Doubao、SiliconFlow、Groq、Ollama 等）。连接原生端点时使用裸 model ID（如 `deepseek-chat`），不得加 `provider/model` 前缀——前缀改写仅在经 OpenRouter 等网关/聚合器路由时生效。HTTP 统一走 rustls-tls 以避免 Windows Schannel TLS 握手失败。