---
kind: error_handling
name: 错误处理：基于标准 Error 与结构化 error 字段的分层策略
category: error_handling
scope:
    - '**'
source_files:
    - apps/cli/src/bin.ts
    - apps/cli/tests/github-webhook-real.e2e.ts
    - apps/cli/tests/fixtures/dsh-badge/snapshot.ts
    - native/landlock-run/README.md
    - snapshots/session/error-finish/headless.snapshot.ts
    - snapshots/session/subagent-published-run-failure/headless.snapshot.ts
---

## 1. 系统/方法概述

该仓库是一个以 TypeScript/Node.js 为主的多包工作区（pnpm workspace），没有引入第三方错误库（如 `ts-error`、`@sentry/*`、`pino` 等）。错误处理采用**分层策略**：

- **业务层**：通过 `throw new Error(...)` / `throw new AggregateError(...)` 抛出原生 `Error`，并约定在错误对象上附加结构化字段（如 `code`、`message`）以便上层统一捕获和序列化。
- **进程边界**：CLI (`apps/cli/src/bin.ts`) 使用 `process.exitCode` + `stderr` 输出错误；headless 模式通过 stdout/stderr 协议将错误帧发送给调用方。
- **网络/IPC 层**：对外暴露的 JSON-RPC / HTTP API 使用 `{ code, message }` 结构体返回错误（见测试中 `envelope.result.error.code` 断言），不直接透传 JS `Error` 堆栈。
- **沙箱/子进程**：`native/landlock-run` 通过 Landlock 限制子进程能力，子进程失败时由宿主读取退出码并转换为诊断信息。

仓库未使用 `try/catch` 包裹所有异步调用的“结果类型”（Result/Either）模式，也未定义统一的自定义 `BaseError` 基类；错误传播主要依赖异常冒泡 + 顶层 catch 集中处理。

## 2. 关键文件与位置

| 位置 | 作用 |
|---|---|
| `apps/cli/src/bin.ts` | CLI 入口，捕获未处理异常并设置 `process.exitCode` |
| `apps/cli/tests/github-webhook-real.e2e.ts` | 展示对外 RPC 错误格式 `{ result: { error: { code, message } } }` |
| `apps/cli/tests/fixtures/dsh-badge/snapshot.ts` | 使用 `AggregateError` 聚合多个清理失败 |
| `native/landlock-run/` | 子进程沙箱启动器，将子进程失败映射为宿主可诊断的错误 |
| `packages/*/src/**/*.ts` | 各业务包普遍使用 `throw new Error(...)` 并附带 `.code` 字段 |
| `scripts/*.ts` | 构建/校验脚本同样遵循 `throw new Error` + 非零退出码的约定 |

## 3. 架构与约定

### 3.1 错误对象约定
- 业务错误通过 `throw new Error(message)` 抛出。
- 常见做法是在错误对象上挂载 `code` 字段（字符串常量或枚举值），例如测试中 `envelope.result.error.code` 表明上游模块会产出带 `code` 的结构化错误。
- 多错误聚合使用原生 `AggregateError`（见 `dsh-badge/snapshot.ts` 中的 cleanup 逻辑）。

### 3.2 进程边界处理
- CLI 入口 (`bin.ts`) 对未捕获异常进行兜底：记录日志后设置 `process.exitCode` 并退出，避免 Node.js 默认打印堆栈到 stdout。
- headless 模式通过 stdout 协议发送错误帧，测试用例断言 `stdout` 关闭前必须收到响应，否则抛错。

### 3.3 网络/IPC 错误序列化
- 对外 JSON-RPC/HTTP 接口统一返回 `{ result: { error: { code, message }, ... } }` 结构，而非直接抛出 JS 异常。这保证了跨语言（Python SDK、浏览器端）的一致性。
- 客户端侧（如 webhook e2e）根据 `error.code` 分支处理不同错误类型。

### 3.4 子进程/沙箱错误
- `native/landlock-run` 作为受限执行环境，子进程崩溃时宿主读取退出码并包装为诊断消息，再向上层暴露。
- 测试覆盖 `partial-landlock-child-failure`、`missing-sandbox-runner` 等场景，验证错误能正确回传到会话层。

## 4. 约束与规范

- **禁止吞掉异常**：测试中大量使用 `if (Date.now() >= deadline) throw new Error(...)` 形式的超时断言，体现“失败即显式抛错”的约定。
- **错误码优先于堆栈**：对外 API 仅暴露 `code` + `message`，内部调试信息保留在日志中，不随响应泄露。
- **无全局错误中间件**：未发现类似 Express/Koa 的错误中间件；错误处理分散在各包的顶层 try/catch 或事件处理器中。
- **测试驱动的错误契约**：`snapshots/` 目录下的 session 快照测试会回放真实运行时的错误路径，确保错误形态稳定（如 `error-finish`、`subagent-published-run-failure` 等场景）。
- **未使用 panic/recover**：TypeScript/JS 环境下无此概念；等价行为是 `throw` + 顶层 catch。

## 5. 总结

该仓库的错误处理体系以**原生 `Error` + 结构化 `code` 字段**为核心，通过**进程边界封装**（CLI exit code、JSON-RPC 错误帧）和**子进程沙箱诊断**实现跨边界传播。设计简洁、无外部依赖，但缺乏统一的错误类型层次和 Result 模式，错误处理逻辑相对分散在各模块的顶层 catch 中。