---
kind: error_handling
name: 错误处理：分层哨兵错误 + JSON-RPC 结构化错误码映射
category: error_handling
scope:
    - '**'
source_files:
    - internal/rpc/protocol.go
    - internal/rpc/control.go
    - internal/storage/contracts.go
    - internal/runtime/budget.go
    - internal/runtime/service.go
    - internal/runtime/context.go
    - internal/runtime/policy.go
    - internal/runtime/runmode.go
    - internal/runtime/sandbox_manager.go
    - internal/runtime/hooks.go
    - internal/runtime/checkpoint.go
    - internal/runtime/tooladapter.go
    - internal/provider/ref.go
    - internal/studio/service.go
    - internal/eval/isolator.go
---

## 1. 整体方案

本仓库采用「分层哨兵错误（sentinel errors）+ 领域结构体错误 + RPC 层统一映射」的模式，没有使用第三方错误库（如 `pkg/errors`、`xerrors`），全部基于 Go 标准库 `errors` 与 `fmt.Errorf`。

- 各内部包在自身边界暴露 `var ErrXxx = errors.New("package: reason")` 形式的哨兵错误，调用方通过 `errors.Is` 判断语义分支。
- 需要携带额外上下文信息的错误定义为具名结构体并实现 `Error()`（部分还实现 `Unwrap()` 以支持 `errors.Is` 链式匹配）。
- 对外暴露面集中在 `internal/rpc` 的 JSON-RPC 控制面，所有业务错误在此被映射为带 JSON-RPC 标准 code 的 `rpc.Error`，客户端只看到 `code/message/data`，不感知 Go error 类型。
- 未发现 `panic/recover` 模式；进程级崩溃恢复由 `Service.Recover` 基于持久化状态（journal / approval row）重建，而非捕获 panic。

## 2. 关键文件与位置

| 层次 | 文件 | 职责 |
|---|---|---|
| RPC 协议 | `internal/rpc/protocol.go` | 定义 JSON-RPC 错误常量（ParseError/InvalidRequest/MethodNotFound/InvalidParams/InternalError/ServerOverload）、`rpc.Error` 结构体、`Handler` 返回 `*Error` |
| RPC 控制面 | `internal/rpc/control.go` | 将下游 `storage.*`、`runtime.*`、`studio.*`、`eval.*` 的错误映射到 JSON-RPC code；提供 `runtimeError` / `studioError` / `internalError` 三个映射函数 |
| 存储契约 | `internal/storage/contracts.go` | 集中声明跨后端共享的哨兵错误：`ErrVersionConflict`、`ErrRunClosed`、`ErrCommitInvalid`、`ErrNotFound`、`ErrLeaseHeld`、`ErrLeaseLost`、`ErrConflict` |
| 运行时 | `internal/runtime/budget.go` | `ErrBudgetExceeded` 哨兵 + `BudgetExceededError` 结构体（含 `Unwrap`） |
| 运行时 | `internal/runtime/service.go` | 审批/问题相关哨兵：`ErrApprovalInvalidDecision`、`ErrQuestionAlreadyAnswered` 等 |
| 运行时 | `internal/runtime/context.go`、`policy.go`、`runmode.go`、`sandbox_manager.go`、`hooks.go`、`checkpoint.go`、`tooladapter.go` | 各类运行时约束错误的哨兵 |
| Provider | `internal/provider/ref.go` | `KeyMissingError` 结构体，用于向调用方暴露“环境变量缺失”的可操作提示 |
| Studio | `internal/studio/service.go` | `ErrAlreadyDecided` 等 studio 层哨兵 |
| Eval | `internal/eval/isolator.go` | `ErrInvalidSuite` 等评估隔离器错误 |

## 3. 架构约定

### 3.1 哨兵错误命名与注释
- 每个包在文件顶部集中声明 `var ( ... )` 哨兵错误，并在注释中说明触发条件（例如 `storage/contracts.go` 对 `ErrVersionConflict`、`ErrRunClosed`、`ErrLeaseLost` 均附有行内注释说明何时返回）。
- 错误消息统一采用 `"package: reason"` 前缀风格（如 `runtime: budget exceeded`、`storage: not found`、`rpc: peer closed`），便于日志检索与分类。

### 3.2 结构化错误 vs 哨兵错误
- 纯语义分支用哨兵：`ErrNotFound`、`ErrBudgetExceeded`、`ErrPeerClosed`、`ErrOverloaded`、`ErrHookBlocked`、`ErrSandboxDenied` 等。
- 需要携带数据时用结构体：`BudgetExceededError{Kind, Limit, Used}`、`KeyMissingError{Provider, EnvKey}`、`rpc.Error{Code, Message, Data}`。其中 `BudgetExceededError` 同时实现 `Unwrap() error` 返回 `ErrBudgetExceeded`，使 `errors.Is(err, ErrBudgetExceeded)` 既能在上层按语义匹配，也能按具体类型匹配。

### 3.3 RPC 层错误映射（唯一对外出口）
`control.go` 中的 `runtimeError` / `studioError` / `internalError` 是业务错误 → JSON-RPC code 的集中映射点：
- `runtimeError`：将 `runtime.ErrInvalidRunMode`、`ErrInvalidPolicyProfile`、`ErrQuestionInvalidAnswer`、`ErrApprovalInvalidDecision`、`ErrApprovalInvalidReason` 映射为 `InvalidParams`；将 `ErrApprovalExpired`、`ErrQuestionExpired`、`ErrRecoveryBusy`、`ErrApprovalAlreadyDecided`、`ErrQuestionAlreadyAnswered` 映射为 `CodeConflict`；将 `ErrApprovalNotFound`、`ErrQuestionNotFound` 映射为 `CodeNotFound`；其余兜底为 `InternalError`。
- `studioError`：将 `studio.ErrInvalid`、`ErrNotHuman`、`eval.ErrInvalidSuite` 映射为 `InvalidParams`；将 `ErrNotReady`、`ErrAlreadyDecided`、`eval.ErrBlockedPath`、`storage.ErrConflict` 映射为 `CodeConflict`；`storage.ErrNotFound` 映射为 `CodeNotFound`。
- `internalError`：任何未识别错误一律返回 `InternalError`，且消息固定为 `"internal error"`，避免泄露内部细节给 UI 客户端。

### 3.4 传输层错误
`rpc.Peer.Serve` 将底层 transport 错误归一化：`io.EOF` → `ErrPeerClosed`；帧过大 → 发送 `InvalidRequest` 错误帧后返回包装错误；出站队列满 → `ErrOverloaded`。`Call` 收到远端 `*Error` 时直接作为 error 返回，调用方可用 `errors.As` 取 `rpc.Error` 读取 code/message/data。

### 3.5 无 panic 策略
代码中未发现 `recover()` 或 `panic()` 的使用；服务重启后的恢复逻辑（`recovery_test.go` 验证）依赖 `Service.Recover` 扫描 journal/approval 行并重建运行状态，而不是捕获 panic 恢复执行流。

## 4. 约定与约束

- **仅 internal/runtime 与 internal/provider 可依赖 eino**，domain 类型不得反向依赖 eino——因此 runtime 层的错误不能引用 eino 的错误类型，必须通过自身的哨兵错误向上冒泡，再由 RPC 层统一映射。
- 所有面向 UI 的错误必须经过 `runtimeError` / `studioError` / `internalError` 映射，禁止 handler 直接返回裸 Go error；这保证了 UI 侧永远看到稳定的 JSON-RPC code 集合。
- 存储后端实现的错误必须返回 `storage/contracts.go` 中声明的哨兵错误（`ErrNotFound`、`ErrConflict`、`ErrVersionConflict` 等），以便上层做跨后端一致的分支判断。
- 结构化错误不得携带用户敏感内容：`BudgetExceededError` 的注释明确限定其仅包含有界计数，不包含 prompt、工具参数、provider 输出或用户内容。
- 错误消息风格统一为 `"package: reason"`，便于运维侧按包前缀过滤日志。
- 测试通过 `errors.Is` 断言哨兵错误（如 `TestServiceRecoverExpiredApprovalFails` 中检查 `cause_category` 间接验证错误路径），体现哨兵错误是跨模块契约的一部分。