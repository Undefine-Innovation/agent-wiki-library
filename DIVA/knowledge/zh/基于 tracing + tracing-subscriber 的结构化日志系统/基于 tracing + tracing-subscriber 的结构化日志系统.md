---
kind: logging_system
name: 基于 tracing + tracing-subscriber 的结构化日志系统
category: logging_system
scope:
    - '**'
source_files:
    - agent-diva-core/src/logging.rs
    - agent-diva-core/src/config/schema.rs
    - agent-diva-cli/src/main.rs
    - agent-diva-gui/src-tauri/src/lib.rs
    - agent-diva-service/src/main.rs
    - agent-diva-agent/src/agent_loop/loop_turn.rs
    - agent-diva-agent/src/agent_loop.rs
---

## 1. 使用的框架与工具

仓库统一采用 **tracing** 作为结构化日志/追踪框架，配合 **tracing-subscriber** 进行订阅层装配、**tracing_appender** 实现文件滚动输出。核心初始化逻辑集中在 `agent-diva-core/src/logging.rs`，通过 `init_logging` / `init_logging_with_terminal_output` 两个入口暴露给 CLI、GUI、Manager 等 crate。

- 日志级别过滤：`EnvFilter`，支持 `RUST_LOG` 环境变量覆盖，并允许从配置中注入模块级 override（如 `module=debug`）。
- 时间格式：RFC 3339 本地时区（`LocalTime::rfc_3339()`），非 UTC。
- 输出字段：target、thread_id、file、line_number 始终开启；JSON 模式下额外启用 JSON 序列化。
- 终端输出：CLI 默认启用 stdout 层，GUI 通过 `init_gui_logging` 调用时传入 `false` 关闭终端输出。
- 文件输出：使用 `tracing_appender::rolling::daily` 按日滚动，固定前缀 `gateway.log`，产出 `gateway.log.YYYY-MM-DD` 命名约定。
- 日志清理：启动后自动扫描目录，删除超过 `retention_days`（默认 30 天）的 `gateway.log*` 与 `gui.log*` 文件；`retention_days=0` 表示保留全部。

## 2. 关键文件与包

- `agent-diva-core/src/logging.rs` — 日志子系统唯一初始化点，包含 EnvFilter 构建、stdout/file 双层装配、旧日志清理。
- `agent-diva-core/src/config/schema.rs` — `LoggingConfig` 结构体定义（`level`、`format`、`dir`、`retention_days`、`overrides: HashMap<String, String>`），由全局配置加载后传入。
- `agent-diva-cli/src/main.rs` — CLI 入口在解析配置后调用 `init_logging_with_terminal_output(&config.logging, enable_terminal_logs)`，是 gateway/agent/chat/tui 等子命令的统一日志起点。
- `agent-diva-gui/src-tauri/src/lib.rs` — GUI 侧 `init_gui_logging` 读取配置、解析 `config.dir` 为绝对路径、以 `terminal=false` 初始化，并将 `WorkerGuard` leak 到进程生命周期。
- `agent-diva-service/src/main.rs` — Windows 服务守护进程使用独立的极简 subscriber（仅 `info` 级别、无 target），不依赖 core 的 logging 模块，用于服务自身状态记录。
- `agent-diva-agent/...` — 业务代码广泛使用 `tracing::{info,warn,error,debug,trace}` 宏，并通过 `trace_id` 字段串联一次 turn 的全链路日志。

## 3. 架构与约定

- **集中式初始化**：所有应用二进制（CLI、GUI、Manager 通过 CLI 间接）都通过 `agent_diva_core::logging` 初始化，避免各 crate 重复配置。
- **配置优先 + 环境变量覆盖**：`RUST_LOG` 可覆盖 `config.level`；`LOG_FORMAT=json|text` 可覆盖 `config.format`；`config.overrides` 提供模块级细粒度覆盖。
- **双 sink 设计**：stdout（可选）+ 每日滚动的文件 sink，两者共享同一 `EnvFilter`，保证控制台与文件输出一致的可筛选性。
- **结构化字段**：日志事件普遍携带 `trace_id`、`step_name`、`loop_index`、`channel`、`sender_id` 等键值对，便于按会话聚合分析（见 `agent-diva-agent/src/agent_loop/loop_turn.rs` 中的 `trace!(trace_id = %trace_id, step_name = ..., ...)`）。
- **跨组件 trace_id 传播**：`trace_id` 作为字符串参数在 agent loop 的各 turn 阶段（admission/context/tool_step/finalize）间传递，并在日志中以 `%trace_id` 形式绑定，形成单条请求的完整追踪链。
- **Span 使用**：在 `agent_loop.rs` 中使用 `tracing::info_span!("AgentSpan", trace_id = %trace_id)` 包裹整个 agent 执行上下文，便于后续接入更丰富的 span 采集。

## 4. 约定与约束

- **日志级别来源顺序**：`EnvFilter::try_from_default_env()` → 失败则回退到 `config.level`；随后叠加 `config.overrides` 中的模块指令。该顺序在 `init_logging_with_terminal_output` 中硬编码实现。
- **文件格式约束**：`LOG_FORMAT` 或 `config.format` 为 `json`（大小写不敏感）时启用 JSON 输出层；否则为人类可读文本格式。
- **文件命名与轮转**：必须使用 `tracing_appender::rolling::daily(dir, "gateway.log")` 前缀，确保产物形如 `gateway.log.2026-08-27`；清理逻辑也仅匹配 `gateway.log*` 和 `gui.log*` 前缀，其他文件不受影响。
- **GUI 日志隔离**：GUI 通过 `resolve_logging_config` 将相对路径解析为相对于 config_dir 的绝对路径后再写入，避免工作目录变化导致日志落盘位置漂移。
- **Windows 服务例外**：`agent-diva-service` 不使用 core 的 logging，而是自建 `tracing_subscriber::fmt()` 且 `with_target(false)`，因为它是系统服务进程，不需要文件输出。
- **测试验证**：`logging.rs` 内嵌单元测试覆盖 retention 默认值（30）、自定义天数、零保留、不存在目录、过期文件删除、非 gateway 文件忽略、gui.log 清理等场景，构成对日志清理行为的契约式约束。
- **日志字段规范**：业务日志应至少包含 `trace_id` 以便跨步骤关联；错误路径使用 `error!` 并附带结构化字段（如 `error_code = error.code()`），而非纯字符串拼接。