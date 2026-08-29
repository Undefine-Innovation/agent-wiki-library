---
kind: external_dependency
name: Supabase 实时数据/认证 SDK（UI 层）
slug: supabase
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
---

React UI 层引入 `@supabase/supabase-js`，用于前端实时数据订阅或认证能力。当前代码中尚未发现直接调用 Supabase 的业务逻辑，可能预留用于未来云端同步或协作功能。若启用需配置 Supabase URL 与 anon key。