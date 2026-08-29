---
kind: error_handling
name: 跨 crate 错误分类与重试体系：thiserror + CategorizeError 统一错误语义
category: error_handling
scope:
    - '**'
source_files:
    - agent-diva-core/src/error.rs
    - agent-diva-core/src/error_category.rs
    - agent-diva-core/src/error_context.rs
    - agent-diva-core/src/security/error.rs
    - agent-diva-providers/src/base.rs
    - agent-diva-providers/src/retry.rs
    - agent-diva-laputa/src/error.rs
    - agent-diva-autodream/src/error.rs
    - agent-diva-sandbox/src/error.rs
    - agent-diva-agent/src/mask/error.rs
    - agent-diva-manager/src/server.rs
---

## 1. 总体方案

Agent Diva 在 Rust workspace 中采用 **per-crate thiserror 枚举 + 跨 crate `CategorizeError` 分类 trait** 的错误处理架构。每个 crate 定义自己的领域错误枚举（`Error` / `LaputaError` / `AutoDreamError` / `SandboxError` / `SecurityError` / `ProviderError` / `MaskError`），通过 `#[derive(Error)]` 和 `#[from]` 自动实现 `std::error::Error`，并在需要时提供 `From<T>` 将第三方错误（`std::io::Error`、`serde_json::Error`、`sqlx::Error`、`reqwest::Error`）转换为自身类型。

跨 crate 的通用错误语义集中在 `agent-diva-core/src/error_category.rs` 中的 `ErrorCategory` 枚举（`Retryable` / `Fatal` / `Config` / `Auth` / `Timeout` / `Unknown`）与 `CategorizeError` trait。所有上层 crate（core、providers、sandbox、security）都实现该 trait，使调用方无需 match 具体错误变体即可决定重试、退避或终止策略。

HTTP 层使用 axum，顶层函数返回 `anyhow::Result<()>`；业务 handler 内部仍返回各自 crate 的强类型 Result，由 axum 默认行为转为 HTTP 响应。网络请求的重试逻辑集中在 `agent-diva-providers/src/retry.rs`，对 5xx 与网络错误做指数退避（基础 1s，最大 3 次重试，±20% jitter），429 直接返回 `ProviderError::RateLimited` 不重试，并支持可选 `RetryListener` 回调上报每次重试。

调试辅助位于 `agent-diva-core/src/error_context.rs`，提供 `ErrorContext` 结构体用于捕获失败操作名、截断后的问题内容（上限 500 字符）与元数据，以及 `find_problematic_chars` 检测控制字符/DEL/替换符等异常字节。

## 2. 关键文件与包

- `agent-diva-core/src/error.rs` — 核心 `Error` 枚举（Config/Io/Serialization/Session/Channel/Provider/Tool/Validation/NotFound/Unauthorized/Internal）及 `From` 转换
- `agent-diva-core/src/error_category.rs` — `ErrorCategory` 与 `CategorizeError` trait，对所有 core/sandbox/security/provider 错误进行分类
- `agent-diva-core/src/error_context.rs` — `ErrorContext` 调试上下文工具
- `agent-diva-core/src/security/error.rs` — `SecurityError`（路径/PII/注入/限流/只读等安全拒绝）
- `agent-diva-providers/src/base.rs` — `ProviderError`（HttpError/JsonError/InvalidResponse/ApiError/ConfigError/RateLimited）及按消息文本推断 Timeout/Retryable/Fatal 的分类逻辑
- `agent-diva-providers/src/retry.rs` — `send_with_retry` 指数退避重试、`classify_response_status`、`parse_retry_after`
- `agent-diva-laputa/src/error.rs` — `LaputaError`（Io/Json/LockTimeout/MemoryStore/Migration 冲突），含稳定 `code()` 机器码
- `agent-diva-autodream/src/error.rs` — `AutoDreamError`（Io/Json/RunNotFound/ActiveRunExists/RunNotCancellable/InvalidState/InputCollection/ProposalPersistence）
- `agent-diva-sandbox/src/error.rs` — `SandboxError`（Denied/ApprovalRequired/PermissionDenied/SpawnFailed/Timeout/ExecutionFailed/Platform* 等）
- `agent-diva-agent/src/mask/error.rs` — `MaskError`（mask 子模块专用）
- `agent-diva-manager/src/server.rs` — axum 路由组装，顶层 `run_server` 返回 `anyhow::Result<()>`

## 3. 架构与约定

- **每 crate 一错枚举**：不使用单一全局错误类型，而是各 crate 暴露自身 `Result<T, XxxError>`，通过 `#[from]` 向上层聚合。
- **分类优先于匹配**：调用方通过 `CategorizeError::category()` 判断是否重试（`Retryable`/`Timeout`）、配置错误（`Config`）、鉴权拒绝（`Auth`）、致命（`Fatal`）或未知（`Unknown`）。例如 `Error::Io(TimedOut)` → `Timeout`，`Error::Unauthorized` → `Auth`，`SecurityError::InjectionDetected` → `Fatal`。
- **Provider 层集中重试**：所有 LLM 调用经 `retry::send_with_retry`，5xx 与网络错误最多重试 3 次，429 立即返回 `RateLimited`，非成功状态走 `ApiError` 分支。
- **安全错误自带用户消息**：`SecurityError` 提供 `user_message()` 与 `is_retryable()`，区分可重试的限流/预算耗尽与不可重试的路径/注入/只读模式。
- **存储错误带机器码**：`LaputaError::code()` 为 HTTP/API 信封提供稳定字符串码（`io`/`json`/`lock_timeout`/`memory_store`/`invalid_memory_migration_id`/`memory_migration_conflict`/`injected_memory_migration_failure`）。
- **调试上下文截断**：`ErrorContext` 限制 `problematic_content` 最长 500 字符，避免日志膨胀。

## 4. 约定与约束

- 所有领域错误必须用 `thiserror::Error` 派生，禁止裸 `String` 错误传播到 crate 边界。
- 新增 crate 若需被上层按“可重试/致命”决策消费，必须实现 `agent_diva_core::error_category::CategorizeError`（sandbox、security、provider 已实现）。
- Provider HTTP 错误分类规则固定：429→`RateLimited`（不重试），5xx→重试，其他非成功→`ApiError`；超时/408/425/5xx 文本中含 timeout 时归类为 `Timeout`。
- 存储类错误（Laputa/AutoDream）对 I/O 使用带 `path` 的 `Io { path, source }` 变体，而非裸 `std::io::Error`，便于定位失败文件。
- 测试覆盖错误分类不变量：`error_category.rs`、`security/error.rs`、`sandbox/error.rs`、`providers/retry.rs` 均包含断言分类结果与 `is_retryable()` 行为的单元测试。
- CLI/Manager 顶层入口使用 `anyhow::Result` 向进程退出码传递错误，但业务 handler 内部保持强类型 Result，不在中间层混用 anyhow。
- 未显式使用 `panic!`/`catch_unwind` 作为错误恢复手段；错误通过 `Result` 上抛，由调用方决定是否记录/重试/终止。