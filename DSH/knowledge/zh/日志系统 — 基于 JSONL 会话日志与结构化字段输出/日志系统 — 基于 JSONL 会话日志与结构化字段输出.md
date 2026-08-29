---
kind: logging_system
name: 日志系统 — 基于 JSONL 会话日志与结构化字段输出
category: logging_system
scope:
    - '**'
source_files:
    - apps/cli/tests/profiles/headless/tests/headless.expected.e2e.ts
    - apps/cli/tests/profiles/headless/tests/compaction.e2e.ts
    - apps/cli/tests/profiles/acp/tests/goal.expected.e2e.ts
    - packages/session-query/session-log-export/src/client/Dialog.tsx
    - packages/session-query/session-log-export/src/client/locales.ts
    - snapshots/headless/headless.snapshot.ts
    - snapshots/acp/acp.snapshot.ts
    - snapshots/sdk/sdk.snapshot.ts
---

## 概述

本仓库没有引入通用的第三方日志框架（如 pino、winston、bunyan 等），也未在业务代码中通过 `console.log` / `console.error` 直接输出调试信息。相反，DeepSeek Harness 的“日志”以**会话级 JSONL 持久化日志**为核心：所有 Agent 运行过程被记录为结构化的 JSON Lines 文件（`session.jsonl` 或等价后端），供 headless 回放、快照测试、ACP/SDK 协议验证以及诊断分析使用。

## 关键文件与位置

- `apps/cli/tests/profiles/headless/tests/*.e2e.ts`：headless 模式下的端到端测试，断言 JSONL 日志中的 `header.parentSession`、`log.content` 等字段，表明日志是测试可观测性的主要来源。
- `apps/cli/tests/profiles/acp/tests/goal.expected.e2e.ts`：通过 `parseJsonl(log.content)` 解析并校验日志内容，说明日志记录由 CLI/Agent 运行时写入，测试侧仅消费。
- `packages/session-query/session-log-export/src/client/Dialog.tsx` 与 `locales.ts`：提供“导出 Session 日志”的 UI 能力，进一步证明日志是以“Session”为单位组织并可导出的资源。
- `snapshots/` 目录：大量按场景组织的 session 快照（`headless.snapshot.ts`、`acp.snapshot.ts`、`sdk.snapshot.ts`），用于断言日志/会话输出的稳定性。

## 架构与约定

1. **日志即会话数据**：日志不是独立于应用状态的旁路输出，而是与 Session 生命周期绑定。每个 Session 对应一份 JSONL 流，包含事件头（如 `header.parentSession`）、工具调用、LLM 交互、文件系统操作等结构化条目。
2. **结构化字段优先**：日志条目使用预定义的结构化字段（如 `header.*`、`content`、`sessionId`、`parentSession`），而非自由文本消息。测试通过解析这些字段进行断言，体现强契约。
3. **多入口统一落盘**：CLI headless、Web、ACP、SDK 等不同运行模式共享同一套日志语义，由底层 Session/Agent 运行时写入，UI 层只负责展示与导出。
4. **无全局 Logger 实例**：代码中未发现 `createLogger`、`logger.info/debug/warn/error` 等通用日志 API；各模块通过事件/状态变更驱动日志条目生成，而不是主动“打点”。
5. **测试依赖日志作为事实源**：E2E 与 snapshot 测试不依赖 stdout/stderr，而是读取已持久化的 JSONL 日志进行断言，因此日志格式稳定是契约的一部分。

## 约束与规则

- 日志格式由测试与快照断言反向约束：任何对 JSONL 结构的修改必须同步更新相关 `*.expected.*` 与 `snapshots/**` 文件，否则 CI 会失败。
- 日志条目需包含足够的上下文字段（如 `sessionId`、`parentSession`）以便构建子代理树与跨会话关联。
- 前端/客户端代码不直接写日志，仅消费 Session 查询与导出能力（见 `session-log-export` 包）。
- 仓库未定义独立的 log level 策略（如 debug/info/warn/error 分级），日志粒度由事件类型决定，而非级别开关。

## 结论

该仓库的“日志系统”本质上是**以 Session 为中心的 JSONL 结构化事件流**，通过 CLI/Agent 运行时统一写入，由测试、快照与导出功能消费。它不依赖外部日志库，也没有传统意义上的 logger 初始化流程；其可靠性由快照测试与 E2E 断言保障。