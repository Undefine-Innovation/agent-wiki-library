---
kind: external_dependency
name: PostgreSQL 持久化后端（可选 Journal 引擎）
slug: postgresql-pgx
category: external_dependency
category_hints:
    - vendor_identity
    - client_constraint
scope:
    - '**'
---

PostgreSQL 是 Vivy 的可选 Journal 存储后端，通过 `github.com/jackc/pgx/v5` 驱动接入，位于 `internal/storage/postgres`。使用方式：设置 `storage.backend: postgres` 并配置 `storage.postgres.dsn_env`（环境变量名），DSN 值由该环境变量注入。同一 DSN 上第二个进程启动会失败（非副本集）。默认 `just docker-up` 走 SQLite（modernc.org/sqlite 嵌入式），`just docker-up-postgres` 是 compose overlay。生产部署时建议用独立 PostgreSQL 实例而非容器内嵌。