---
kind: logging_system
name: 基于标准库 log/slog 的 JSON 结构化日志系统
category: logging_system
scope:
    - '**'
source_files:
    - cmd/vivy/main.go
    - internal/app/app.go
    - internal/events/bus.go
    - internal/runtime/audit.go
    - internal/runtime/approval_scheduler.go
---

## 1. 使用的系统与框架

仓库使用 Go 标准库 `log/slog`（Go 1.21+）作为唯一生产日志框架，通过 `slog.NewJSONHandler(os.Stdout, nil)` 将日志以 JSON Lines 格式输出到标准输出。进程启动时（`cmd/vivy/main.go`）创建全局默认 logger 并调用 `slog.SetDefault`，后续所有包通过 `slog.Default()` 或注入的 `*slog.Logger` 获取实例。

没有引入任何第三方日志库（如 zap、logrus、zerolog），也没有自定义日志中间件或统一包装器。

## 2. 关键文件与位置

- `cmd/vivy/main.go`：进程入口，安装 JSON handler 到 stdout，设置全局默认 logger；在 worker 模式分支中刻意跳过该安装以避免污染 worker 协议 stdout。
- `internal/app/app.go`：应用装配层持有 `logger *slog.Logger` 字段，构造时通过 `slog.Default()` 获取，并在启动/关闭/配置覆盖等关键路径记录 Info/Warn/Error。
- `internal/events/bus.go`：事件总线在订阅者掉队时通过 `slog.Warn` 记录带 `run`、`seq` 字段的告警。
- `internal/runtime/audit.go`：定义 `SlogAuditSink`，仅记录稳定元数据（RunID、Seq、Type、PayloadBytes、PayloadSHA256），明确注释“payload 本身不跨越此边界”，避免泄露工具参数或密钥。
- `internal/runtime/approval_scheduler.go`：遗留使用 `log.Logger`（标准库 `log` 包）而非 slog，通过构造函数注入，未接入 JSON handler。

## 3. 架构与约定

### 初始化策略
- 单点安装：仅在 `cmd/vivy/main.go` 中创建 `slog.NewJSONHandler(os.Stdout, nil)` 并设为全局默认。
- 依赖注入：`App` 结构体显式持有 `*slog.Logger` 字段，其他组件通过函数参数传入（如 `applySettingsOverlay(ctx, logger, cfg)`）。
- 回退策略：`SlogAuditSink.Record` 中若传入的 Logger 为 nil，则回退到 `slog.Default()`。

### 结构化字段约定
日志条目通过 slog 的键值对形式携带上下文，常见字段包括：
- `err`：错误对象（Error 级别）
- `addr`、`path`、`provider`、`model`：配置相关
- `run`、`seq`：运行时事件追踪
- `payload_bytes`、`payload_sha256`：审计元数据（不含 payload 内容）

### 日志级别使用
- `Error`：启动失败、组合失败、运行失败、配置验证失败等不可恢复错误。
- `Warn`：可恢复异常（如 settings overlay 应用失败、VIVY_ADDR 覆盖 server.addr、订阅者掉队）。
- `Info`：正常生命周期事件（服务启动/关闭、settings overlay 生效）。
- `Debug`：审计钩子（`vivy audit event`），用于外部审计消费。

### 特殊约束
- Worker 模式分支在 logger 安装之前处理，确保 worker 协议的 stdout 不被 JSON 日志污染（见 main.go 注释：“The worker protocol owns stdout”）。
- 审计日志严格只记录 payload 的 size 和 SHA256 摘要，不记录原始负载，防止敏感信息泄露。

## 4. 约定与约束

**已观察到的实践：**
- 所有新代码应通过 `slog` 输出结构化 JSON 日志到 stdout。
- 业务组件不应直接操作 `os.Stdout` 写日志，而应使用注入或默认的 `*slog.Logger`。
- 审计相关日志必须遵循 `SlogAuditSink` 的“仅元数据”原则，不得包含完整 payload。

**不一致之处（待清理）：**
- `internal/runtime/approval_scheduler.go` 仍使用旧式 `log.Logger`（标准库 `log` 包）并通过 `Printf/Println` 输出纯文本，未接入 JSON handler，与其他模块的 slog 风格不一致。该文件是历史遗留，尚未迁移。

**无以下机制：**
- 没有按模块/包前缀过滤日志级别。
- 没有环境变量控制日志级别（如 LOG_LEVEL）。
- 没有多 sink（同时输出到文件、stdout、远程收集器）。
- 没有统一的日志上下文（如请求 ID、trace ID）贯穿所有日志条目。