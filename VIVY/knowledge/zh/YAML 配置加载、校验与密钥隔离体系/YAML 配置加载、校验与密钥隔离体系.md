---
kind: configuration_system
name: YAML 配置加载、校验与密钥隔离体系
category: configuration_system
scope:
    - '**'
source_files:
    - internal/config/config.go
    - internal/config/config_test.go
    - cmd/vivy/main.go
    - config.example.yaml
    - config.dev.yaml
    - docker/config.yaml
    - docker/config.postgres.yaml
    - internal/app/app.go
---

## 1. 系统概览

Vivy 使用基于 `gopkg.in/yaml.v3` 的**强类型 YAML 配置 + 启动时严格校验**方案。配置入口为 `internal/config/config.go`，提供 `Default()` 内置默认值、`Load(path)` 从文件解码并合并到默认值之上、`Validate()` 集中校验所有字段；进程入口 `cmd/vivy/main.go` 负责选择配置文件路径并在启动前完成全部校验，非法配置直接中止进程。

## 2. 关键文件

- `internal/config/config.go` — 类型定义 (`Config`, `Server`, `Storage`, `Providers`, `Runtime`, `Tools`, `Governance`)、`Default()`/`Load()`/`Validate()`、secret 边界正则 `envKeyPattern`。
- `config.example.yaml` — 完整配置模板，注释即文档。
- `config.dev.yaml` — 开发覆盖层（仅开启 `runtime.mock`）。
- `docker/config.yaml` / `docker/config.postgres.yaml` — 容器覆盖层，分别挂载 SQLite 和 Postgres 后端。
- `cmd/vivy/main.go` — 解析 `VIVY_CONFIG` 环境变量 → 回退到工作目录 `config.yaml` → 再回退到 `config.Default()`；支持 `VIVY_ADDR` 运行时覆盖监听地址并重新 `Validate()`。
- `internal/app/app.go` — 装配根：读取已验证的 `cfg`，按 `storage.backend` 打开存储，按 `providers.active` 加载 bundle，将 `tools.enabled` 注册到工具集，将 `governance.profiles` 注入策略引擎，将 `sandbox.*` 注入沙箱管理器。

## 3. 架构与设计决策

### 3.1 分层加载顺序
1. `config.Default()` 提供完整的内置默认值（含 `server.addr=127.0.0.1:8787`、`storage.backend=sqlite`、`providers.active=openai`、`runtime.workspace_root=data/workspaces`、`runtime.skills_root=data/skills`、`tools.enabled` 全量白名单、`governance.profiles` 四个预置 profile）。
2. 可选 YAML 文件通过 `yaml.NewDecoder(...).KnownFields(true)` **严格解码**，未知字段直接报错，防止拼写错误被静默忽略。
3. 多个覆盖层叠加：`config.dev.yaml` 用于本地离线开发，`docker/config*.yaml` 用于容器部署，均只声明差异项。
4. 进程级环境变量覆盖：`VIVY_CONFIG` 指定配置文件路径（存在时不回落工作目录），`VIVY_ADDR` 覆盖监听地址并触发二次 `Validate()`。
5. 运行时设置覆盖：`internal/app/settings` 在 `app.New` 中读取独立 agent working dir 下的 settings 文件，非侵入地覆盖 `providers.active`、`default_model`、`base_url`（通过 `os.Setenv` 走 provider 现有机制）。

### 3.2 Secret 隔离（D-010）
- Provider 密钥**永远不出现在配置文件中**。`Provider.EnvKey` 仅保存环境变量名（匹配 `^[A-Z][A-Z_]*$`），实际 key 在模型构造时从环境读取。
- `Postgres.DSNEnv` 同理，DSN 字符串不得内联进 YAML。
- `MCPServer.AuthEnv` 遵循相同约定。
- 若 YAML 中出现非 schema 定义的 secret 字段（如 `api_key:`），`KnownFields(true)` 会拒绝；若 `env_key` 被误填为字面密钥，`Validate()` 用正则拒绝。
- 测试 `TestSecretInjectionRejected` 显式断言这两种注入方式均失败。

### 3.3 安全约束与白名单
- `server.allowed_origins` 仅允许 loopback 主机，且当 `server.addr` 为非 loopback（如 `0.0.0.0`）时必须为空（由 `validateServerOrigins` 强制）。
- `runtime.http_allowed_hosts` 默认仅 `localhost/127.0.0.1/::1`。
- `runtime.execute_allowed_commands` 默认仅 `go/git/rg`。
- `runtime.sandbox.default_mode` 限定为 `read_only` / `workspace_write` / `danger_full_access` 三档；`approval.default_policy` 限定为 `ask` / `never` / `auto`。
- `runtime.mock_scenario` 限定为 `hitl|approval|question|timeout|stale`。
- `governance.profile` 限定为 `default|plan|read_only|full_auto`；规则中的 `field` 限定为 `command|cmd|path|filepath|file_path`；`decision` 限定为 `allow|prompt|deny`。

### 3.4 资源上限与熔断
`Runtime` 中大量 `max_*` 字段（`stream_buffer`、`max_event_payload_bytes`、`max_tool_turns`、`max_context_bytes`、`max_history_messages`、`max_tool_result_bytes`、`max_run_events`、`max_model_calls`、`max_run_tool_calls`、`max_run_retries`、`http_max_response_bytes`）在 `Validate()` 中强制为正数或合理范围，作为 NFR 的硬上限。

### 3.5 数据目录推导
`Config.DataDirectory()` 根据 `storage.data_dir`、`storage.backend`、`storage.sqlite.path` 推导 settings/evals 的工作目录，SQLite 模式默认取 db 所在目录，Postgres 模式固定为 `data`。

## 4. 约定与约束

- **配置文件位置优先级**：`VIVY_CONFIG` > 工作目录 `config.yaml` > 内置默认值（带 warning）。`VIVY_CONFIG` 存在时绝不回退。
- **Secret 永不落盘**：provider key、Postgres DSN、MCP auth 仅以环境变量名引用，运行时按需读取。
- **未知字段即错误**：启用 `KnownFields(true)`，任何不在 schema 中的键都会导致启动失败。
- **覆盖层只声明差异**：`config.dev.yaml`、`docker/config.yaml`、`docker/config.postgres.yaml` 都只覆盖必要字段，其余继承自 `Default()`。
- **启动失败即退出**：`loadConfig` 与 `app.New` 中任何配置相关错误都直接 `os.Exit(1)`，无降级运行。
- **运行时覆盖需重新校验**：`VIVY_ADDR` 覆盖后调用 `cfg.Validate()`，确保 loopback/origin 策略不被绕过。
- **Settings overlay 容错**：settings 文件损坏仅记录 warning 并跳过，不影响主流程；但生产 config 必须合法。
- **Compose 端口绑定契约**：测试断言 `docker-compose.yml` 必须以 `127.0.0.1:8787:8787` 绑定宿主环回地址，禁止暴露到所有接口。

## 5. 适用性说明

该配置系统贯穿 `cmd/vivy`、`internal/app`、`internal/config`、`internal/runtime`、`internal/provider`、`internal/storage`、`internal/tools`、`internal/studio` 等模块，是进程装配的单一事实来源，并通过 `config.example.yaml`、`config.dev.yaml`、`docker/*.yaml` 以及 `internal/config/config_test.go` 中的大量用例共同维护契约。
