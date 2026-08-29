---
kind: external_dependency
name: BML 记忆存储：SQLite + FTS5
slug: sqlite-sqlx
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
source_files:
    - Cargo.toml
    - README.md
---

BML（Basic Memory Layer）以 profile-local 的 SQLite + FTS5 数据库（`.laputa/memory.sqlite3`）作为唯一生产级记忆权威，通过 SQLx（runtime-tokio-rustls, sqlite, migrate, chrono features）访问；旧版文件仅作为离线导入源。Laputa 治理层建立在 BML 之上，所有写入必须经 governed apply 路径。