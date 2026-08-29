---
kind: configuration_system
name: Agent Diva 配置系统：JSON + 环境变量分层加载与热重载
category: configuration_system
scope:
    - '**'
source_files:
    - agent-diva-core/src/config/mod.rs
    - agent-diva-core/src/config/loader.rs
    - agent-diva-core/src/config/schema.rs
    - agent-diva-core/src/config/validate.rs
    - agent-diva-cli/src/main.rs
---

## 1. 系统概览

Agent Diva 的配置系统集中在 `agent-diva-core/src/config/`，由三个文件组成：
- `schema.rs`：定义所有配置结构体（`Config`、`AgentsConfig`、`ChannelsConfig`、`ProvidersConfig`、`ToolsConfig`、`GatewayConfig`、`SandboxConfig`、`MateConfig`、`LoggingConfig` 等），全部基于 `serde` 的 `Serialize`/`Deserialize`，使用 `#[serde(default)]` 提供默认值。
- `loader.rs`：实现 `ConfigLoader`，负责从 JSON 配置文件与环境变量合并、别名归一化、验证、持久化以及基于 mtime 的后台热重载。
- `validate.rs`：集中业务级校验规则（如 temperature 范围、MCP server 必须提供 command 或 url、web search provider 白名单、mate TTS/ASR provider 白名单等）。

CLI (`agent-diva-cli`) 通过 `CliRuntime` 暴露 `--config` / `--config-dir` / `-w/--workspace` 参数，将配置路径传给 `ConfigLoader::with_file` / `with_dir`；其余 crate（agent loop、autodream、files 等）直接依赖 `agent_diva_core::config`。GUI (Tauri) 通过 Manager HTTP API 间接读写同一份 `~/.agent-diva/config.json`。

## 2. 配置来源与优先级

加载顺序（`load()` 中严格串行执行）：
1. **默认值**：`Config::default()` 生成完整默认树。
2. **JSON 配置文件**：读取 `config_path`（默认 `~/.agent-diva/config.json`），用 `merge_values` 递归合并到默认树上。
3. **别名覆盖**：`apply_alias_overrides` 把一组固定环境变量映射到 `providers.<name>.api_key`（如 `ANTHROPIC_API_KEY` → `providers.anthropic.api_key`，共 11 个 provider key）。
4. **路径式环境变量**：`apply_path_overrides` 扫描所有以 `AGENT_DIVA__` 开头的环境变量，按 `__` 分割为小写段并写入对应 JSON 路径（例如 `AGENT_DIVA__AGENTS__DEFAULTS__MODEL=openai/gpt-4o` 设置 `agents.defaults.model`）。
5. **键名归一化**：`normalize_alias_keys` 把历史/兼容字段合并到规范键（`channels.neuro_link` 同时接受 `neuro_link`/`generic_pipe`；`tools.mcpServers`/`tools.mcp_servers`；顶层 `pet` 重命名为 `mate`）。
6. **反序列化为强类型**：`serde_json::from_value::<Config>`。
7. **校验**：调用 `validate_config(&config)`，失败则返回 `Error::Validation(...)`。

优先级从高到低：**路径式环境变量 > 别名环境变量 > JSON 文件 > 默认值**。测试覆盖了该顺序（`test_load_applies_alias_env_overrides`、`test_load_applies_path_env_overrides`、`test_path_env_overrides_alias_and_file`）。

## 3. 热重载机制

`ConfigLoader::start_hot_reload(on_diff)` 启动一个 tokio 任务，每 5 秒检查 `config_path` 的 mtime，检测到变化后等待 500ms 防抖再重新 load，并用 `compute_diff(old, new)` 计算差异。差异通过 `classify_field(path)` 分类：
- **hot_reload 允许列表**：`agents.defaults.model|temperature|max_tool_iterations|reasoning_effort`、`tools.budget.*`、`tools.exec.timeout`、`tools.web.search.enabled`、`tools.web.fetch.enabled`、`sandbox.mode|approval_policy`、`logging.level`。
- 其他所有字段（provider、channel、gateway port、MCP server 等）均归类为 **restart_required**。

调用方根据 `diff.has_restart_required()` 决定是否重启服务。该机制仅用于运行时观察与决策，不自动应用变更。

## 4. 配置目录约定

- 默认配置目录：`dirs::home_dir().join(".agent-diva")`，配置文件名为 `config.json`。
- CLI 支持 `--config <file>` 和 `--config-dir <dir>` 覆盖默认位置。
- 工作区数据统一放在 `<workspace>/.agent-diva/` 下（token ledger、security policy、Laputa 状态、Autodream 报告等），与用户配置分离。
- GUI/Tauri 的 `src-tauri/tauri.conf.json` 是打包配置，不属于运行时应用配置。

## 5. 关键数据结构摘要

| 配置段 | 主要用途 | 默认行为 |
|---|---|---|
| `agents.defaults` | 默认 provider/model/max_tokens/temperature/max_tool_iterations/reasoning_effort/thinking_mode | deepseek-chat，max_tokens=8192，temp=0.7 |
| `channels.*` | Telegram/Discord/WhatsApp/Feishu/DingTalk/Email/Slack/QQ/Matrix/IRC/Mattermost/Nextcloud Talk/Neuro-link | 全部 disabled，各自有连接凭据字段 |
| `providers.*` | 每个 LLM 厂商的 api_key/api_base/custom_models/response_protocol | 空字符串 |
| `tools.*` | 内置工具开关、Web 搜索/抓取、Exec 超时、MCP servers、context compaction budget | 多数工具默认 enabled，cron/planning 默认 false |
| `gateway` | host/port | 0.0.0.0:3000 |
| `sandbox` | 沙箱模式、Windows 隔离级别、网络访问、审批策略、保护路径 | WorkspaceWrite |
| `mate` | 桌面虚拟人开关、VRM 模型、TTS/ASR 提供商及密钥、语速音量 | enabled=true，tts_provider="browser" |
| `logging` | level/format/dir/overrides/retention_days | info/text/logs/30天 |
| `reports.llm_curation` | 节奏报告是否走 LLM 编排 | enabled=true，fallback="deterministic" |
| `self_evolution` | AutoDream 触发频率、阈值、auto_merge_confidence | disabled |
| `memory.l1_index_lines` | 注入 system prompt 的索引行数 | 30 |

## 6. 约束与校验规则

- `agents.defaults.workspace` 不能为空。
- `max_tokens`、`max_tool_iterations` 必须 > 0。
- `temperature` 必须在 [0.0, 2.0]。
- `reasoning_effort` 必须是 `low`/`medium`/`high` 之一。
- 每个 `tools.mcp_servers.<name>` 必须至少设置 `command`(stdio) 或 `url`(http)。
- `tools.web.search.provider` 仅限 `brave`/`bocha`/`zhipu`，且 `max_results` 上限随 provider 不同（zhipu/bocha 为 50，其他为 10）。
- `mate.asr_provider` 仅限 `web_speech`/`siliconflow`；`tts_provider` 仅限 `browser`/`openai`/`siliconflow`/`minimax`；`tts_speed`>0，`tts_volume`∈[0.0, 2.0]。
- `reports.llm_curation.fallback` 当前仅支持 `deterministic`。

## 7. 与其他组件的关系

- `agent-diva-agent` 在 agent loop 初始化时通过 `ConfigLoader::new()` 加载配置，并将 `~/.agent-diva/workspace` 作为工作区根。
- `agent-diva-manager` 通过 `GatewayRuntimeConfig` 接收已解析的 `Config` 实例，由 CLI 启动 gateway 时传入。
- `agent-diva-cli` 的 `ConfigCommands`（path/init/refresh/validate/doctor/show）直接调用 `ConfigLoader` 与 `validate_config`，并提供 `--json` 输出。
- `agent-diva-files`、`agent-diva-autodream`、`agent-diva-laputa` 等 crate 通过约定路径 `<workspace>/.agent-diva/...` 共享运行时数据，但不直接依赖配置加载逻辑。

## 8. 设计要点

- **纯 JSON 持久化 + 环境变量注入**：没有 YAML/INI/`.env` 文件，所有可序列化配置都落在单一 `config.json`，敏感信息可通过环境变量注入而不污染文件。
- **向后兼容键别名**：通过 `serde(alias=...)` 与 `normalize_alias_keys` 逐步迁移旧键（`pet→mate`、`mcpServers→mcp_servers`、`neuro_link→neuro-link`）。
- **显式 allowlist 的热重载分类**：只有白名单字段可在运行时无重启生效，其余一律要求重启，避免隐式半更新状态。
- **全量 diff 而非增量 patch**：`compute_diff` 将两个 `Config` 序列化为 JSON Value 后逐路径比较，便于 UI 展示 before/after 值。
- **无 feature flag 框架**：功能开关通过配置字段（如 `tools.cron`、`tools.planning`、`channels.*.enabled`）控制，没有编译期 feature gate 式的配置层。