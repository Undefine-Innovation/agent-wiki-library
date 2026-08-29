---
kind: dependency_management
name: Go/Node 双栈依赖管理：单模块 + npm 镜像 + Git Submodule
category: dependency_management
scope:
    - '**'
source_files:
    - go.mod
    - go.sum
    - ui/package.json
    - ui/pnpm-lock.yaml
    - ui/.npmrc
    - .gitmodules
    - sdk/main.go
    - studio/dsh-vivy-studio/package.json
---

## 1. 使用的系统与方法

本仓库采用 **Go 单模块 + Node.js 子项目** 的双栈依赖管理模式：
- Go 侧：根目录 `go.mod` / `go.sum` 声明单一 module `agent-vivy`，使用 `require` 显式列出所有直接依赖与间接依赖（无 vendor 目录）。
- Node 侧：`ui/package.json` + `ui/pnpm-lock.yaml` 管理前端依赖；`studio/dsh-vivy-studio/package.json` 是独立的轻量主题包，不引入运行时依赖。
- 外部代码通过 **Git Submodule** 引入：`.gitmodules` 将 `.agents/skills/oil-frontend` 作为只读参考源码纳入版本控制。
- 私有注册表：未配置 GOPRIVATE、GOPROXY 或私有 Go 模块代理；Node 侧通过 `ui/.npmrc` 将 npm registry 指向国内镜像 `https://registry.npmmirror.com/`。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `go.mod` | 定义 module `agent-vivy`、Go 版本 `1.26.4`、全部 direct/indirect 依赖 |
| `go.sum` | Go 依赖校验摘要（由 go mod 生成） |
| `ui/package.json` | 前端运行时与开发依赖清单（React 19、Vite 7、Radix UI、TanStack Router/Query、Zustand、Tailwind v4 等） |
| `ui/pnpm-lock.yaml` | pnpm 锁定的精确依赖树 |
| `ui/.npmrc` | 指定 npm 源为 npmmirror 镜像 |
| `studio/dsh-vivy-studio/package.json` | DSH web shell 的主题/品牌包，仅声明元数据与静态资源 |
| `.gitmodules` | 声明 `oil-frontend` 子模块来源 |
| `sdk/main.go` | `vivy-sdk` 独立 CLI 入口，复用同一 Go module 的 `sdk/internal` 包 |

## 3. 架构与约定

### Go 依赖边界约束
作者指南明确要求：**仅 `internal/runtime` 与 `internal/provider` 可依赖 `github.com/cloudwego/eino*`，domain 类型不得反向依赖 eino**。该约束通过 Go 包的 import 路径隔离实现——`internal/domain`、`internal/storage`、`internal/rpc`、`internal/tools` 等包均不 import eino，从而在编译期强制领域层与编排层的解耦。eino 及其扩展（`eino-ext/components/model/openai`、`eino-ext/libs/acl/openai`）被集中在 runtime/provider 层消费。

### 存储后端抽象
依赖选择体现“多后端”设计：SQLite 通过 `modernc.org/sqlite`（纯 Go 实现，无需 CGO）提供嵌入式后端，PostgreSQL 通过 `jackc/pgx/v5` 提供持久化后端，二者统一由 `internal/storage` 契约暴露，便于切换。

### SDK 与主模块共享依赖
`sdk` 目录不是独立 Go module，而是根 module 下的子包（`agent-vivy/sdk/internal`），因此 `vivy-sdk` CLI 与主进程共享同一份 `go.mod`，避免依赖版本漂移。

### Node 前端依赖策略
- 使用 pnpm 锁定依赖树（`pnpm-lock.yaml`），确保构建可重现。
- 通过 `.npmrc` 全局切换至 npmmirror，加速国内拉取。
- 组件库基于 Radix UI + Tailwind CSS v4 + shadcn 风格，状态管理用 Zustand，路由/查询用 TanStack Router/Query。
- `studio/dsh-vivy-studio` 是独立 npm 包但无运行时依赖，仅打包静态资源供 DSH shell 加载。

### 外部代码引入方式
`oil-frontend` 以 Git Submodule 形式引入到 `.agents/skills/oil-frontend`，作为只读参考实现，不参与主模块编译。

## 4. 约定与约束

- **Go 模块范围**：整个仓库是一个 Go module，不存在 `vendor/` 目录，依赖通过 `go.mod` + `go.sum` 管理，需联网解析。
- **eino 依赖隔离**：只有 `internal/runtime` 和 `internal/provider` 能 import `github.com/cloudwego/eino*`；其他 internal 包禁止反向依赖，违反将在编译时报错。
- **Go 版本固定**：`go.mod` 顶部声明 `go 1.26.4`，要求构建环境使用该版本。
- **Node 包管理器**：前端使用 pnpm（由 `pnpm-lock.yaml` 推断），安装时需启用对应 registry 镜像。
- **Submodule 只读**：`.agents/skills/oil-frontend` 通过 submodule 引用外部仓库，不应修改其内容。
- **无私有 Go 代理**：未发现 GOPRIVATE、GOPROXY 或自定义 Go 模块代理配置，依赖均来自公共 Go 模块代理。
- **SDK 复用模块**：`vivy-sdk` 命令与主程序共享依赖，新增依赖需在根 `go.mod` 中声明，避免 SDK 与运行时版本不一致。
