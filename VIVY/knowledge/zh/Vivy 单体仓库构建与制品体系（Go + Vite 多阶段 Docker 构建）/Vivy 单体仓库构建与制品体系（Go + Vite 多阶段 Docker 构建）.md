---
kind: build_system
name: Vivy 单体仓库构建与制品体系（Go + Vite 多阶段 Docker 构建）
category: build_system
scope:
    - '**'
source_files:
    - justfile
    - Dockerfile
    - docker-compose.yml
    - docker-compose.postgres.yml
    - go.mod
    - internal/buildinfo/buildinfo.go
    - ui/package.json
    - ui/vite.config.ts
    - ui/embed.go
    - dev.ps1
    - sdk/main.go
---

## 1. 使用的构建系统/工具
- Go 模块：单模块 `agent-vivy`，Go 版本锁定在 `go.mod` 的 `go 1.26.4`。
- 任务编排：使用 `justfile` 作为统一入口（Windows 下通过 `powershell.exe -NoProfile -Command` 执行），替代 Makefile。
- UI 构建：`ui/` 子目录为独立 pnpm + Vite 项目（`package.json`、`vite.config.ts`、`pnpm-lock.yaml`），产物输出到 `ui/dist`。
- 容器化：多阶段 Dockerfile（`node:22-bookworm-slim` 构建前端 → `golang:1.26-bookworm` 编译 Go → `debian:bookworm-slim` 运行）。
- 开发辅助：PowerShell 脚本 `dev.ps1` 启动 split-loop（后端 `./cmd/vivy` :8787 + Vite :3015），`docker-compose.yml` / `docker-compose.postgres.yml` 组合服务。

## 2. 关键文件
- `justfile`：所有构建/测试/打包任务的单一入口（setup、build、test、vet、fmt-check、ci、headless-compile、build-split、sdk、studio、docker-build、docker-up、docker-up-postgres、test-postgres、ui-e2e、run、dev）。
- `Dockerfile`：三阶段镜像构建，将 Vite 产物嵌入后由 Go 二进制内嵌。
- `docker-compose.yml` / `docker-compose.postgres.yml`：默认 SQLite 模式与可选 Postgres 叠加。
- `internal/buildinfo/buildinfo.go`：通过 `-ldflags -X agent-vivy/internal/buildinfo.Version=...` 注入版本号。
- `ui/vite.config.ts`：固定 dev 端口 3015、strictPort、outDir `dist`、assetsDir `assets`，并代理 `/rpc` 到 `http://127.0.0.1:8787`。
- `ui/embed.go`：使用 `//go:embed all:dist` 将生产构建内嵌进二进制；配合 `vivy_headless` build tag 切换 headless 模式。
- `sdk/main.go`：独立的 `vivy-sdk` 可执行文件，用于校验/打包用户插件。
- `dev.ps1`：Windows 上启动 split-loop 开发环境，自动检测端口占用、安装依赖、启动后端与 Vite。

## 3. 架构与约定
- **单体进程 + 内嵌 UI**：`./cmd/vivy` 是主物种进程，生产构建时通过 `//go:embed all:dist` 把 `ui/dist` 静态嵌入，运行时以 HTTP handler 暴露前端，无需外部 Web 服务器。
- **Split 开发 vs 嵌入式发布**：
  - 开发：`just dev` 或 `dev.ps1` 同时跑后端 (:8787) 和 Vite HMR (:3015)，Vite 通过 proxy 转发 `/rpc` 到后端。
  - 发布：`just build-split` 先 `go build -tags vivy_headless -o dist/vivy-backend.exe ./cmd/vivy`，再 `pnpm build -- --outDir ../dist/vivy-ui --emptyOutDir`，产出分离的后端与 UI 目录；Dockerfile 则直接内嵌 UI。
- **Headless 构建标签**：`headless-compile` 使用 `go test -run '^$' -tags vivy_headless ./cmd/vivy ./ui` 验证无 UI 嵌入时的编译；`embed.go` 用 `//go:build !vivy_headless` 控制是否 embed。
- **GOPROXY/NPM_REGISTRY 镜像**：`just setup`、`Dockerfile`、`dev.ps1`、`docker-compose.yml` 均强制使用 `https://goproxy.cn,direct` 与 `https://registry.npmmirror.com`，保证国内网络可构建。
- **版本注入**：`internal/buildinfo.Version` 默认 `"dev"`，release 构建通过 `-ldflags "-X agent-vivy/internal/buildinfo.Version=<version>"` 覆盖。
- **健康检查**：容器与 compose 均通过 `wget http://127.0.0.1:8787/healthz` 探测存活。
- **数据持久化**：默认 SQLite，卷挂载 `/data`；Postgres 通过 `docker-compose.postgres.yml` 叠加。

## 4. 约定与约束
- **仅一个 Go 模块**：整个仓库只有一个 `go.mod`，所有包（`cmd/*`、`internal/*`、`sdk/*`、`plugins/*`）共享同一依赖图。
- **Eino 依赖约束**：按作者指南，仅 `internal/runtime` 与 `internal/provider` 可依赖 `github.com/cloudwego/eino*`，domain 类型不得反向依赖 eino（此约束由代码组织体现，非编译器强制）。
- **UI 端口硬约束**：`vite.config.ts` 注释明确“dev server 必须监听 3015 + strictPort”，且 `dev.ps1` 会主动检测端口冲突并报错。
- **安全约束**：compose 注释强调“Do not publish 8787 on all host interfaces”，端口绑定到 `127.0.0.1:8787`，因为 `/rpc/bootstrap` 无用户认证。
- **镜像只读数据**：容器内 `/data` 通过 volume 持久化，二进制与配置从镜像层只读拷贝。
- **CI 流水线等价命令**：`just ci` 串联 `fmt-check`、`vet`、`test`、`headless-compile`、`ui-ci`（pnpm install → typecheck → test → build），可作为 CI 入口。
- **SDK 与 Studio 是独立二进制**：`just sdk` 产出 `vivy-sdk.exe`，`just studio` 产出 `vivy-studio.exe`，与日常运行的 `vivy` 物种进程解耦。
- **测试隔离**：Postgres 测试需显式设置 `VIVY_POSTGRES_TEST_DSN`，否则 `test-postgres` 跳过；SQLite 测试随 `go test ./...` 默认运行。