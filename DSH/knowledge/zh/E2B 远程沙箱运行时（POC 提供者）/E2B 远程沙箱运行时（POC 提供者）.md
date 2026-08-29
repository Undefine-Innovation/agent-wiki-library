---
kind: external_dependency
name: E2B 远程沙箱运行时（POC 提供者）
slug: e2b
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
source_files:
    - packages/e2b/README.md
    - packages/README.md
---

### E2B
- 角色：为 dsh 提供远程代码执行沙箱的后端实现，属于 sandbox 能力家族的一部分，当前标记为 POC。
- 集成方式：通过 `packages/e2b` 暴露 sandbox、FS/subprocess 适配器等 provider，被 shell、code-runtime、subprocess 等消费方通过 capability seam 注入使用。
- 状态：按 packages/README.md 的发布预期，`e2b/` 组属于 POC，兼容性期望低于产品级 group。